# OrderFlow — referência do backend para o frontend

---

## 0. Como usar este documento

- **Para quem:** a sessão que vai planejar e construir o frontend (SPA no navegador, provavelmente Vite em `http://localhost:5173`) sem acesso à conversa que produziu isto. O documento cobre só o que o frontend precisa para consumir a API.
- **Fonte da verdade, nesta ordem:**
  1. o código no repositório `P:\Portifolio\orderFlow`;
  2. o OpenAPI servido pelo gateway: `GET http://localhost:8080/v3/api-docs/{identity|catalog|cart|orders|payment|sales|notifications}`, com índice em `/v3/api-docs/swagger-config` e UI em `/swagger-ui.html`.
  
  Quando o OpenAPI e o código divergem, vale o código. As divergências conhecidas estão na §3.10. O `openapi.yaml` da raiz do repo é um export antigo do monólito (sem `/cart`): **não use**.
- **Commit de referência:** `14587bc` ("docs: conclui as tarefas das notificações"), branch `docs/conclui-notificacoes`.
- **Como foi verificado:** leitura de controllers, DTOs, entidades, services, `application.yml`, migrações, testes (`*Test`, `*IT`, `e2e/**/*E2E.java`) e README. Comportamento de biblioteca foi conferido no bytecode das versões empacotadas (Spring Boot 4.1.1, Spring Security 7.1.1, spring-data-commons 4.1.1, jackson-databind 3.1.5). Na redação e na revisão (2026-11-18) a stack local estava parada (`localhost:8080` recusou conexão), então o OpenAPI **não** foi relido ao vivo. A revisão usou a captura dos 7 documentos feita pelo gateway local em 2026-11-17 15:19, depois do commit de referência (10:04, sem mudança de código desde então). Ela tem os mesmos 27 paths e 32 operações da §5.0 e confirma o que a §3.10 diz do spec. O resto vem das anotações springdoc.
- **Marcação:** "(inferido)" = deduzido do código ou da biblioteca, sem teste no repo. Tudo o que não foi confirmado está na §11.
- **Caminhos citados** são relativos à raiz do repo. `.../` abrevia `src/main/java/com/matheus/orderFlow/<pacote>/`.
- **Manutenção:** atualize este documento quando mudar controller, DTO, regra de domínio, rota ou CORS do gateway, ou configuração que o front sente (timeouts, paginação, TTLs). Troque também o commit de referência.
- **Vocabulário:** "dono" = o `sub` do token é o dono do recurso. "Autenticado" = qualquer token válido (USER ou ADMIN). "Fato" = evento publicado no RabbitMQ.

---

## 1. Visão geral

### 1.1 Arquitetura

```
navegador (SPA, http://localhost:5173)
   │  HTTP/JSON + Authorization: Bearer <JWT>
   ▼
gateway :8080   Spring Cloud Gateway Server WebMVC: roteia, CORS, timeouts. NÃO valida token.
   ├─ /auth/**, /users/**       → identity       MySQL (usuários) + Redis (refresh tokens). Único emissor de JWT.
   ├─ /products/**              → catalog        MySQL (produtos)
   ├─ /cart/**                  → cart           Redis (carrinhos) ──HTTP──► catalog, orders
   ├─ /orders/**                → orders         MySQL (pedidos + outbox) ──HTTP──► catalog
   ├─ /payments/**              → payment        MySQL (cobranças) ──HTTP──► gateway de pagamento externo (WireMock local)
   ├─ /admin/sales/**           → sales          MongoDB (projeção order_views) ──HTTP──► orders (só no rebuild)
   └─ /admin/notifications/**   → notifications  MySQL (notificações, contatos) ──SMTP──► Mailpit; ──HTTP──► identity (só no import)

RabbitMQ (exchange orderflow.events): orders e identity publicam fatos pelo outbox;
payment, sales e notifications consomem.
```

Fontes: `gateway/src/main/resources/application.yml`, `docker-compose.yml`, `README.md` (§Arquitetura).

| Serviço | O que possui | Publica | Consome |
|---|---|---|---|
| identity | contas, hash de senha (BCrypt), papel, refresh tokens, chave de assinatura | `user.registered`, `user.email-changed` | — |
| catalog | produtos | nada (`services/catalog/pom.xml` sem mensageria) | — |
| cart | um carrinho por usuário (TTL 7 dias) | — | — (chama catalog e orders por HTTP) |
| orders | pedidos, itens (nome e preço congelados), máquina de estados | `order.confirmed`, `order.shipped`, `order.delivered`, `order.cancelled` (`contracts/.../order/OrderFacts.java`) | — |
| payment | cobranças | nada | `order.confirmed` |
| sales | projeção para relatórios | — | os 4 fatos de pedido |
| notifications | cópia do e-mail de cada usuário, notificações por e-mail | — | fatos de pedido e de usuário |

- Referência entre serviços é por UUID, sem chave estrangeira. Cada dado só sai pela API do serviço dono. **Não existe rota agregada** (pedido com pagamento, pedido com cliente).
- O gateway repassa `Authorization`, método, path, query e corpo intactos. Os 401 e 403 vêm de cada serviço (`GatewayRoutingIT.shouldForwardMethodPathBodyAndAuthorizationUntouched`; `gateway/pom.xml` sem `platform-security`).

### 1.2 Consistência eventual

- **Imediato (síncrono):** tudo o que o serviço dono lê do próprio banco. Exemplos: o status de um pedido depois de `confirm`, `ship` etc. (`GET /orders` lê o MySQL), o carrinho, o produto, a conta.
- **Eventual:**
  - o pagamento, que nasce depois de `POST /orders/{id}/confirm`;
  - os relatórios de vendas;
  - os e-mails;
  - a cópia do e-mail do cliente no notifications.
- **Mecanismo:** cada mudança grava o fato no **outbox** na mesma transação. Um relay publica com atraso fixo de `OUTBOX_POLL_INTERVAL` entre rodadas (padrão 1000 ms, até 100 mensagens por rodada) (`platform/platform-outbox/.../OutboxRelay.java`; `services/orders/src/main/resources/application.yml`).
- **Não há push:** nenhum WebSocket, SSE ou `SseEmitter` no código. O front descobre o que é assíncrono por **polling** ou recarregando.

**O que já aparece logo depois de cada passo** (o front não precisa esperar evento para ler isto):

| Depois de | Já reflete na resposta e no próximo GET | Só depois, por evento |
|---|---|---|
| `POST /cart/checkout` 200 ou `POST /orders` 201 | o pedido `PENDING` em `GET /orders` e `GET /orders/{id}` (mesmo MySQL); no checkout, `GET /cart` passa a dar 404 | nada: criar pedido não publica fato (`OrderService.createOrder` não chama `publishEvent`) |
| `POST /orders/{id}/confirm` | `status: CONFIRMED` | pagamento (`GET /payments/orders/{id}` dá 404 até o fato chegar), e-mail, relatórios |
| `ship`, `deliver`, `cancel` | o novo `status` | e-mail, relatórios |
| `PUT /users/{id}/email` | `GET /users/{id}` e o próximo login | o destino dos e-mails de pedido (cópia no notifications) |

Fonte: `services/orders/.../order/OrderService.java` (só `confirmOrder`, `shipOrder`, `deliverOrder` e `cancelOrder` publicam), `services/identity/.../user/UserService.java`.

---

## 2. Rodar localmente

### 2.1 Comandos

| Objetivo | Comando |
|---|---|
| Subir tudo | `docker compose up --build` |
| Conferir | `curl http://localhost:8080/actuator/health` → `"status":"UP"` (só diz que o **gateway** está de pé) |
| Aplicar nova `CORS_ALLOWED_ORIGINS` | definir a variável no ambiente e `docker compose up -d gateway` |
| Só a infraestrutura | `docker compose up -d mysql redis rabbitmq` (o README lista o resto) |

- O Compose só sobe o gateway depois que todos os serviços ficam *healthy* (`depends_on ... condition: service_healthy`). Se um serviço cair depois, as rotas dele dão 503 e o health continua `UP`.
- **Modo Maven** (serviços pelo `./mvnw`, infra no Compose, README linhas 263-290): a 8080 é o serviço **orders direto, sem gateway**. Identity fica na 8083, catalog na 8081 e cart na 8082. Não há CORS nesse modo. Para o front, use o Compose completo.
- **Volume antigo** (dados do monólito): os passos de migração estão no README, linhas 157-261. Volume novo não precisa de nada.

### 2.2 Portas e UIs (`docker-compose.yml`)

| Serviço | Host | Uso |
|---|---|---|
| gateway | **8080** | API, Swagger UI (`http://localhost:8080/swagger-ui.html`) e OpenAPI |
| mailpit | 8025 | caixa de entrada dos e-mails: `http://localhost:8025`. O SMTP 1025 fica só na rede interna |
| zipkin | 9411 | rastros: `http://localhost:9411` |
| rabbitmq | 5672, 15672 | broker e painel (`http://localhost:15672`) |
| wiremock | 8090 → 8080 | gateway de pagamento falso. Uma única mapping, que **sempre aprova** (`wiremock/mappings/charge-approved.json`) |
| mysql | 3308 → 3306 | schemas por serviço |
| redis | 6379 | carrinhos e refresh tokens |
| mongo | 27017 | projeção de vendas |
| identity, catalog, cart, orders, payment, sales, notifications | nenhuma | 8080 só dentro da rede `orderflow-network` |

O front só fala com a 8080.

### 2.3 Administrador em desenvolvimento

`services/identity/.../user/AdminBootstrap.java`, um `ApplicationRunner`, roda a cada subida do identity:

- Se já existe algum `ADMIN`, não faz nada. Nunca cria um segundo (`UserIT#shouldNotCreateASecondAdministrator`).
- Senão, cria um admin com `ADMIN_NAME`, `ADMIN_EMAIL` e `ADMIN_PASSWORD`:
  - As três têm padrão em `services/identity/src/main/resources/application.yml`.
  - O Compose repassa **só** `ADMIN_EMAIL` e `ADMIN_PASSWORD`, com padrões definidos no próprio `docker-compose.yml` (serviço `identity`). `ADMIN_NAME` definida no host não tem efeito sem editar o Compose.
  - Com e-mail ou senha vazios, só registra aviso no log.
  - Se o e-mail já pertence a um USER comum, não cria nada e avisa no log. Não há promoção.
  - A senha passa pela política (8 caracteres ou mais). Se for curta, o identity **não sobe**.
  - O admin é criado sem gravar `user.registered`.
- O admin entra pelo mesmo `POST /auth/login` e recebe `roles: ["ADMIN"]`.
- **Não existe rota** para promover usuário, listar usuários ou trocar papel.

Outras variáveis que o front sente. Nenhuma é repassada pelo Compose, a não ser `CORS_ALLOWED_ORIGINS`, então no Compose valem os padrões:

| Variável | Padrão | Efeito |
|---|---|---|
| `CORS_ALLOWED_ORIGINS` (gateway) | `http://localhost:5173` | origens aceitas, separadas por vírgula |
| `JWT_EXPIRATION` | `15m` | validade do access token (`expiresIn: 900`) |
| `JWT_REFRESH_EXPIRATION` | `7d` | validade do refresh, renovada a cada rotação |
| `JWT_PRIVATE_KEY` | ausente no Compose | sem ela, a chave de assinatura muda a cada reinício do identity (§4.8) |
| `OUTBOX_POLL_INTERVAL` | 1000 ms | atraso do outbox |
| `NOTIFICATIONS_POLL_INTERVAL` | 5000 ms | intervalo do envio de e-mails |
| `MAIL_FROM` | definido no `application.yml` do notifications | remetente dos e-mails |

### 2.4 Carga de contatos (uma vez)

O notifications guarda uma cópia do e-mail de cada usuário, alimentada pelos fatos `user.registered` e `user.email-changed`. Usuários que existiam antes do serviço (ou o admin, que nasce sem fato) não estão nessa cópia. Um ADMIN chama **uma vez** `POST /admin/notifications/contacts/import` (§5.7). A chamada é idempotente: rodar de novo sem novidades devolve `{"imported": 0}`.

### 2.5 Testes e2e e mocks

- **API do Mailpit** (`services/notifications/src/test/.../integration/AbstractIntegrationTest.java`):
  - `GET /api/v1/search?query=to:<email>` → `$.messages[*].ID`
  - `GET /api/v1/message/{ID}` → `$.Subject`, `$.Text`
  - `DELETE /api/v1/messages` apaga **todos** os e-mails.
  
  Chame pelo runner (Node ou Playwright), não de dentro da SPA: o CORS do Mailpit não foi verificado.
- **Pagamento local:** o WireMock aprova tudo. Para as telas de `DECLINED`, `FAILED`, `PENDING` e do 404 do pagamento, use mocks no front (por exemplo MSW).

---

## 3. Convenções HTTP

### 3.1 Base URL e roteamento

- **Base única:** `http://localhost:8080`. O path pelo gateway é **o mesmo** do serviço, sem prefixo.

| Prefixo (casa também sem sufixo) | Serviço |
|---|---|
| `/auth/**`, `/users/**` | identity |
| `/products/**` | catalog |
| `/cart/**` | cart |
| `/orders/**` | orders |
| `/payments/**` | payment |
| `/admin/sales/**` | sales |
| `/admin/notifications/**` | notifications |
| `/v3/api-docs/{serviço}` | o serviço, com o path reescrito para `/v3/api-docs` |

- **Não há rota padrão.** Qualquer outro caminho dá **404** no próprio gateway, com `{"status":404,"message":"Could not find: /<caminho>","errors":{}}` (`gateway/.../UpstreamErrorHandling.java#handleUnroutedPath`). Isso inclui `/internal/**`, `/.well-known/jwks.json`, `/actuator/env` e `/admin` sozinho (`GatewayRoutingIT.shouldNotExposeRoutesThatStayInsideTheNetwork`).
- Cada rota usa `PreserveHostHeader`, então o `Location` sai com o `Host` que o cliente usou.
- O gateway **nunca repete** uma requisição (`gateway/.../UpstreamClientConfig.java`).

### 3.2 CORS (`gateway/.../CorsConfig.java`)

| Item | Valor |
|---|---|
| Origens | lista exata de `CORS_ALLOWED_ORIGINS`, separada por vírgula (padrão `http://localhost:5173`). `allowedOrigins`, não padrões: `http://localhost:*` não funciona |
| Métodos | `GET`, `POST`, `PUT`, `DELETE`. **Sem `PATCH` e sem `HEAD`** |
| Headers de requisição | só `Authorization`, `Content-Type`, `Idempotency-Key` |
| Headers expostos | só `Location` |
| Credenciais | desligadas: sem cookies, sem `Access-Control-Allow-Credentials` |
| Max-age | não configurado. Vale o padrão do Spring, 1800 s |
| Preflight | respondido no gateway (200), sem chegar ao serviço (`GatewayRoutingIT.shouldAnswerTheBrowserPreflightWithoutReachingTheService`) |
| Origem, método ou header fora da lista | preflight **403**, sem `Access-Control-Allow-Origin` (`GatewayRoutingIT.shouldRefuseThePreflightFromAnUnknownOrigin`). O corpo é o texto `Invalid CORS request` do `DefaultCorsProcessor` do Spring, não `ErrorResponse` (inferido: o teste só confere status e header) |

- Alternativa sem CORS: o proxy do dev server (`server.proxy` do Vite) deixa as chamadas same-origin. Nesse caso o `Location` sai com o `Host` que o proxy repassar (depende de `changeOrigin`).
- O `/actuator/health` usa outro handler mapping, sem esse CORS. Ler o health cross-origin provavelmente é bloqueado (inferido).

### 3.3 Headers

| Header | Direção | Uso |
|---|---|---|
| `Authorization: Bearer <token>` | requisição | toda rota não pública. **Não envie** nas rotas públicas (§4.1) |
| `Content-Type: application/json` | requisição | obrigatório em POST e PUT **com corpo**. Rotas de ação sem corpo (`confirm`, `cancel`, `ship`, `deliver`, `checkout`, `rebuild`, `import`) não precisam |
| `Idempotency-Key` | requisição | só `POST /orders` lê (§3.8) |
| `Location` | resposta | URL absoluta nos 201 de `POST /auth/register`, `POST /products` e `POST /orders` |

- Não existem `ETag`, `X-Trace-Id`, `Retry-After` nem rate limiting (nenhum 429 no código).
- O 401 não traz `WWW-Authenticate` (inferido de `SecurityErrorHandlers.java`).
- O backend aceita `traceparent`, mas o CORS não deixa o navegador enviá-lo.

### 3.4 `ErrorResponse`

```json
{ "status": 400, "message": "price: Product price cannot be negative", "errors": { "price": "Product price cannot be negative" } }
```

Fonte: `platform/platform-web/.../exception/ErrorResponse.java`, `GlobalExceptionHandler.java`; `platform/platform-security/.../SecurityErrorHandlers.java`.

- `status` (int) repete o status HTTP. `message` vem **em inglês**. `errors` (objeto `string → string`) **sempre** vem: `{}` quando não há campo a culpar, nunca `null`.
- **Um campo por vez:** a validação do domínio para no primeiro erro, então `errors` tem no máximo uma chave. Nenhum DTO tem Bean Validation.
- Nos erros de campo, `message` = `"<campo>: <texto>"`.
- Um 4xx de um serviço chamado por outro (cart → orders, orders → catalog, sales → orders, notifications → identity) chega **com o mesmo status e corpo** (`handleCalledServiceError`).

| Origem | Status | `message` | `errors` |
|---|---|---|---|
| regra de domínio | 400 | `<campo>: <texto>` | `{ "<campo>": "<texto>" }` |
| JSON malformado, corpo ausente, tipo errado, `null` em campo primitivo | 400 | `Invalid request body` | `{}` |
| path ou query com tipo inválido (UUID, enum, data, número) | 400 | `<p>: Parameter '<p>' has an invalid value` | `{ "<p>": … }` |
| parâmetro obrigatório ausente | 400 | `<p>: Parameter '<p>' is required` | `{ "<p>": … }` |
| `sort` em campo desconhecido | 400 | `sort: Cannot sort by unknown field '<x>'` | `{ "sort": … }` |
| sem token, ou token inválido, expirado, adulterado ou de outra chave | 401 | `Authentication is required to access this resource` | `{}` |
| login inválido | 401 | `Invalid credentials` | `{}` |
| refresh inválido | 401 | `Invalid refresh token` | `{}` |
| papel insuficiente, ou `/users/{id}` alheio (`@PreAuthorize`) | 403 | `You do not have permission to perform this operation` | `{}` |
| não existe, ou não é seu | 404 | `Could not find: <id>` | `{}` |
| estado impede, ou duplicidade | 409 | mensagem da regra | `{}` |
| concorrência (`@Version`) | 409 | `Resource was modified by another request, retry the operation` | `{}` |
| exceção não tratada | 500 | `Internal Server Error` | `{}` |
| dependência interna fora ou lenta | 503 | `Service unavailable: catalog` / `orders` / `identity` | `{}` |
| gateway não conecta ao serviço (recusa ou mais de 1 s) | 503 | `Service unavailable` | `{}` |
| serviço não responde em 4,5 s | 504 | `Service did not respond in time` | `{}` |
| path sem rota no gateway | 404 | `Could not find: /<caminho>` | `{}` |

- O `SecurityErrorHandlers` tem outro 403 (`...to access this resource`), mas ele só dispara por regra de papel na cadeia de filtros, e nenhuma rota tem essa regra. Na prática todo 403 de serviço vem com `...to perform this operation`.
- **Sem handler próprio, e por isso provavelmente 500** (inferido): método não mapeado (405), `Content-Type` ausente ou não JSON em rota com corpo (415), subpath inexistente dentro de um prefixo roteado (ex.: `/orders/{id}/foo`). Todos caem no `@ExceptionHandler(Exception.class)`.

### 3.5 Status nesta API

| Status | Significado aqui |
|---|---|
| 200 | sucesso, inclusive ações (`confirm` etc.), `PUT /users/{id}/password` e `POST /cart/checkout` |
| 201 | criação com `Location`: cadastro, produto, `POST /orders` (inclusive repetição com a mesma chave) |
| 204 | sem corpo: `DELETE /products/{id}`, `POST /auth/logout`. Não chame `response.json()` |
| 400 | cliente mandou algo inválido |
| 401 | sem token ou token inválido/expirado; credencial ou refresh inválidos |
| 403 | papel insuficiente. Também `/users/{id}` de outra conta |
| 404 | não existe **ou não é seu** (pedido, pagamento). Também "ainda não existe" (pagamento, carrinho) |
| 409 | estado impede (transição, item indisponível no checkout, e-mail em uso, concorrência) |
| 500 | erro genérico. Veja os casos conhecidos acima |
| 503 | serviço ou dependência fora, ou circuit breaker aberto |
| 504 | gateway desistiu em 4,5 s. **A operação pode ter acontecido** |

### 3.6 Paginação: `PageResponse<T>`

`{ content: T[], page: int, size: int, totalElements: int64, totalPages: int }` (`platform/platform-web/.../web/PageResponse.java`).

- `page` começa em **0**.
- `size` é o tamanho de página **aplicado**, já com teto. Não é a contagem de `content`.
- Não há `first`, `last` nem `hasNext`: use `page + 1 < totalPages`.
- Página além da última → **200** com `content: []`.
- Query: `page`, `size`, `sort=campo,asc|desc` (repetível). Mandar `sort` **substitui** o padrão inteiro.
- Normalização silenciosa (spring-data-commons 4.1.1, conferido no bytecode, sem teste): `page` negativo ou não numérico vira 0; `size < 1` ou não numérico vira o padrão; `size` acima do teto vira o teto.

| Rota | Padrão | Teto de `size` | Campos de `sort` |
|---|---|---|---|
| `GET /products` | 20, `createdAt,DESC` | 100 (`ProductIT.shouldCapThePageSizeAtTheConfiguredMaximum`) | `id`, `name`, `description`, `price`, `createdAt`, `updatedAt` |
| `GET /orders` | 20, `createdAt,DESC` | 100 | `id`, `userId`, `idempotencyKey`, `status`, `total`, `createdAt`, `updatedAt`, `version` |
| `GET /admin/notifications` | 20, `updatedAt,DESC` (`NotificationController.java`) | sem `max-page-size` no yml. Vale o padrão do Boot, 2000 (inferido) | propriedades de `Notification` (`updatedAt`, `createdAt`, `attempts`, `orderStatus`...) |

- Para paginação estável, inclua desempate: `sort=price,asc&sort=id,asc`.
- `sort=status` em pedidos ordena pelo nome do enum (alfabético: `CANCELLED`, `CONFIRMED`, `DELIVERED`, `PENDING`, `SHIPPED`).
- `GET /admin/sales/daily` e `/top-products` devolvem **array simples**, sem envelope.

### 3.7 Formatos no JSON

Nenhum `application.yml` define `spring.jackson.*`: valem os padrões do Boot 4.1.1 com jackson-databind 3.1.5.

| Tipo | Formato |
|---|---|
| UUID | string canônica. Path inválido → 400 com `errors.<param>` |
| `Instant` (`createdAt`, `updatedAt`) | ISO-8601 em UTC, terminado em `Z`. Número **variável** de casas na fração: o orders trunca em microssegundos (`services/orders/.../order/JpaAuditingConfig.java`), os outros não. Todas as colunas são `TIMESTAMP(6)` (migrações `V1__*.sql` de cada serviço), então o que é relido do banco tem no máximo 6 casas, e a resposta de um POST/PUT, montada em memória, pode ter mais. Use parser ISO completo |
| `LocalDate` (sales: `date`, `from`, `to`) | `YYYY-MM-DD` |
| Dinheiro (`price`, `total`, `totalPrice`, `unitPrice`, `subtotal`, `amount`) | **número** JSON (BigDecimal). Banco `DECIMAL(12,2)`. Sem campo de moeda: o sistema é BRL (`PaymentGatewayClient.CURRENCY`; e-mail em `R$`). `POST/PUT /products` devolve o preço **como enviado** (`120` continua `120`) |
| Enums | string MAIÚSCULA |
| Campo desconhecido na entrada | **ignorado em silêncio** (`FAIL_ON_UNKNOWN_PROPERTIES` desligado pelo Boot). `role`, `price` ou `status` enviados pelo cliente não têm efeito |
| `null` em campo primitivo (`int`) | 400 `Invalid request body` (`FAIL_ON_NULL_FOR_PRIMITIVES` ligado por padrão no jackson-databind 3.1.5, conferido no bytecode) |
| Decimal em campo inteiro (ex.: `quantity: 1.5`) | provavelmente truncado (`ACCEPT_FLOAT_AS_INT` ligado por padrão, conferido no bytecode; efeito sem teste) |
| `null` na saída | provavelmente `null` explícito (sem `@JsonInclude` no repo; não visto ao vivo). Trate `null` e ausente igual |

### 3.8 Idempotência e repetição

| Operação | Repetir é seguro? | Como |
|---|---|---|
| `POST /orders` | sim, **com** `Idempotency-Key` | gere um UUID por intenção de compra e reaproveite só nas repetições. Até 64 caracteres, único **por usuário**. A mesma chave devolve **201** com o pedido original, **ignorando o conteúdo** do corpo da repetição e sem chamar o catálogo. O corpo ainda precisa ser JSON legível: ausente ou malformado dá 400 `Invalid request body` antes da busca pela chave (`@RequestBody` obrigatório em `OrderController.createOrder`). Só a **ausência** do header desliga a idempotência: header vazio ou `"undefined"` vira uma chave fixa (inferido) (`OrderService.createOrder`, `Order.java`) |
| `POST /cart/checkout` | sim, sem editar o carrinho | o cart gera e guarda a própria chave (`Cart.startCheckout`). O front **não** envia `Idempotency-Key` |
| `POST /admin/sales/rebuild`, `POST /admin/notifications/contacts/import` | sim | idempotentes e guardam o que já gravaram |
| `POST /cart/items` | **não** | soma à quantidade |
| `PUT /cart/items/{id}` | sim | define a quantidade |
| transições de pedido | **não** | a segunda dá 409 |
| `POST /auth/refresh` | **nunca** | o reuso derruba a sessão (§4.5) |

### 3.9 Prazos, falhas e circuit breakers

| Trecho | Prazo | Fonte |
|---|---|---|
| gateway → serviço | conexão 1 s (→ 503), resposta 4,5 s (→ 504) | `gateway/src/main/resources/application.yml` |
| checkout, no total | 4 s, conferidos **entre** etapas (não aborta a chamada em curso) | `services/cart/.../application.yml` (`orderflow.checkout.deadline`) |
| cart → catalog, orders → catalog | conexão 1,5 s, resposta 1,5 s, **uma chamada por item, em sequência** | yml de cart e orders |
| cart → orders | conexão 1,5 s, resposta 3,5 s | yml do cart |
| sales → orders (rebuild), notifications → identity (import) | 10 s por página, sem prazo total | yml de sales e notifications |
| payment → gateway de pagamento | conexão 2 s, resposta 3 s, 3 tentativas (espera 0,5 s e 1 s) | yml do payment |

- **Circuit breakers** de cart→catalog, orders→catalog e cart→orders: janela de 10 chamadas, mínimo de 5, abre com 50% de falha, **10 s aberto**, 2 chamadas de teste. Aberto, responde 503 na hora. 4xx não conta como falha. São **compartilhados por todos os usuários**.
- **Circuit breaker do payment → gateway de pagamento:** mesma janela, **30 s aberto**.
- Quando um container volta com outro IP, o gateway pode dar 503 por **até 30 s** (cache de DNS, README linhas 1230-1233).

### 3.10 OpenAPI: o que ele não diz

Conferido na captura de 2026-11-17 (§0): os 7 documentos são `openapi: 3.1.0`, com `servers[0].url = http://localhost:8080`, `security: [{bearerAuth: []}]` no topo, nenhum `required` em schema e nenhum `nullable`. As 32 operações da §5.0 estão todas lá, e nenhuma outra.

- `bearerAuth` está declarado **globalmente**, inclusive nas rotas públicas (`@OpenAPIDefinition(security=...)` em cada `OpenApiConfig.java`). Um gerador de cliente pode injetar o header onde não deve (§9.2).
- O media type das respostas aparece como `*/*` (nenhum `produces`). O corpo real é JSON.
- Nenhum schema marca `required` nem `nullable`. Os tipos gerados saem todos opcionais, e `gatewayReference`, `reason` e `lastError` saem como `string`, sem `| null`.
- **Não documentados:**
  - o 401 de token, em todas as rotas protegidas;
  - o 400 de UUID inválido e o 400 `Invalid request body`;
  - o 403 de `POST/PUT/DELETE /products`;
  - o 503 das rotas do carrinho fora do checkout, e o do `POST /orders` (só no texto do `info`);
  - o 500 de senha longa;
  - a regex do e-mail;
  - o 400 de `sort` em notifications;
  - o 504 do gateway.
- `POST /cart/checkout` aparece como `JsonNode` (propriedades internas do Jackson). Na prática devolve um `OrderResponse` (`CartController.checkout`).
- Em notifications, `Pageable` sem `@ParameterObject` aparece como um parâmetro `pageable` (objeto, obrigatório). Na prática são `page`, `size` e `sort` na query.
- O payment diz que `FAILED` "may be retried", e o README diz "pode ser reprocessado". **O código não implementa isso** (§6.2).
- `/internal/**` e `/.well-known/**` ficam fora do OpenAPI e do gateway.

Conclusão: use os tipos da §5.9, escritos a partir do código, em vez de gerar cegamente pelo spec.

---

## 4. Autenticação e sessão

Fontes: `services/identity/.../user/{AuthController,UserController,UserService,TokenService,RefreshTokenService,RefreshToken,JwtKeyConfig,User}.java`; `platform/platform-security/.../{SecurityConfig,SecurityErrorHandlers,JwkSetDecoderAutoConfiguration,AuthenticatedUser}.java`.

### 4.1 O que é público

| Público | Fonte |
|---|---|
| `/auth/**` | `services/identity/src/main/resources/application.yml` (`orderflow.security.public-paths`) |
| `GET /products`, `GET /products/{id}` | `services/catalog/src/main/resources/application.yml` (`public-get-paths`) |
| `/swagger-ui/**`, `/swagger-ui.html`, `/v3/api-docs/**`, `/actuator/health` | `SecurityConfig.ALWAYS_PUBLIC` |

Todo o resto exige token (`anyRequest().authenticated()`). Sessão `STATELESS`, CSRF desligado.

**"Público" quer dizer que o token não é exigido, não que é ignorado.** O `oauth2ResourceServer` vale para todas as rotas e não há `BearerTokenResolver` customizado (`SecurityConfig.java`). Um `Authorization: Bearer` inválido ou expirado, inclusive `Bearer undefined`, leva **401** antes do controller, mesmo em `/auth/**` e `GET /products` (comportamento do Spring Security 7.1.1, sem teste no repo). Não envie `Authorization` nessas rotas.

### 4.2 O que o backend entrega

| Item | Formato | Onde vem | Validade |
|---|---|---|---|
| access token | JWT RS256 | campo **`token`** (não `accessToken`) do corpo JSON | `JWT_EXPIRATION`, padrão 15 min. `expiresIn` em segundos (900) |
| refresh token | string opaca, 43 caracteres `[A-Za-z0-9_-]` (32 bytes base64url) | campo `refreshToken` do corpo JSON | `JWT_REFRESH_EXPIRATION`, padrão 7 dias, renovado a cada rotação |

`TokenResponse = { token: string, type: "Bearer", expiresIn: int64, refreshToken: string }`.

- **Não há cookie**, e o CORS não libera credenciais. Não use `credentials: 'include'` nem `withCredentials`.
- **Onde guardar** (memória, `sessionStorage`, `localStorage`) é decisão do front. O backend só entrega os dois tokens no corpo e espera o access no header `Authorization`.
- O refresh fica no Redis só como hash SHA-256 (`RefreshTokenIT#shouldNotStoreTheRefreshTokenAsIs`).

### 4.3 Cadastro e login

- `POST /auth/register` → **201** com `UserResponse` (`id`, `name`, `email`, `createdAt`, `updatedAt`). **Não devolve token nem papel.** O papel é sempre `USER`: não existe campo `role`, e um enviado é ignorado. Chame `POST /auth/login` em seguida.
- `POST /auth/login` → 200 `TokenResponse`. **Não devolve dados do usuário.**
- E-mail inexistente e senha errada respondem **igual**, `401 Invalid credentials`, inclusive no tempo: a senha é conferida contra um hash descartável (`UserService.login`, `prepareAbsentUserHash`; `UserIT#shouldAnswerTheSameForUnknownEmailAndWrongPassword`). Pelo código, campo vazio ou ausente também dá 401.
- O BCrypt (custo 12, `orderflow.security.bcrypt-strength` em `IssuerConfig.java`) faz login, cadastro e troca de senha levarem **centenas de ms**.

### 4.4 Access token (JWT)

| Claim | Conteúdo |
|---|---|
| `iss` | `"orderflow"` |
| `sub` | id do usuário (UUID em string) |
| `roles` | array com **um** valor, sem prefixo: `["USER"]` ou `["ADMIN"]` |
| `iat`, `exp` | segundos desde a época |

- Header JWT com `alg: RS256` e `kid`. Não há `aud`, `jti`, nome nem e-mail (`UserIT#shouldNotCarryPersonalDataInTheToken`). O payload é só codificado (base64url), não cifrado.
- **"Quem sou eu":** não existe `/users/me`. Decodifique o payload: `sub` dá o id, `roles` dá o papel. `UserResponse` não tem `role`. Os dados de perfil vêm de `GET /users/{sub}`.
- Os serviços prefixam `ROLE_` internamente. A claim reflete o papel **na emissão**. O refresh relê o papel do banco (`RefreshTokenIT#shouldCarryTheCurrentRoleOnRefresh`).
- Para renovar antes de expirar, use `expiresIn` contado a partir do recebimento, ou `exp`.
- Os serviços aceitam o token até **60 s depois do `exp`**: o `NimbusJwtDecoder` usa `JwtValidators.createDefault()`, cujo `JwtTimestampValidator` tem folga de 60 s (Spring Security 7.1.1, conferido no bytecode). Nenhum serviço troca o validador (`JwkSetDecoderAutoConfiguration.java`, `JwtKeyConfig.java`). Não conte com essa folga: renove antes.

### 4.5 Refresh: rotação e detecção de reuso

- `POST /auth/refresh` com `{ refreshToken }` → 200 com **novo** `token` **e novo** `refreshToken`. O refresh enviado é marcado `used`.
- **Família = um login.** Cada `POST /auth/login` abre uma família nova. Logins em dispositivos diferentes são independentes (`RefreshTokenIT#shouldKeepSeparateLoginsIndependent`).
- **Reuso derruba a família:** se chega um refresh já usado, todos os tokens daquele login são apagados e a resposta é 401. O refresh legítimo mais novo **também** passa a dar 401 (`RefreshTokenIT#shouldRevokeTheWholeFamilyWhenARotatedTokenIsReused`).
- **Concorrência:** `rotate` busca, confere e salva sem trava. Dois refresh **simultâneos** com o mesmo token têm desfecho não determinístico.
- **Sessão deslizante:** cada rotação dá 7 dias cheios ao token novo. Não há prazo absoluto.
- 401 `Invalid refresh token` cobre: desconhecido, expirado, vazio/nulo, já usado, revogado por logout, usuário que não existe mais.

**Consequências obrigatórias para o front:**
1. **Refresh single-flight:** no máximo um `POST /auth/refresh` em voo (promise compartilhada), coordenado entre abas (Web Locks ou BroadcastChannel). Duas abas que renovam com o mesmo refresh deslogam o usuário.
2. Depois de cada refresh, troque **os dois** tokens guardados.
3. **Nunca repita automaticamente** um refresh que pode ter chegado ao servidor (504, timeout, erro de rede). O identity pode ter rotacionado e a resposta se perdido. Reenviar o token antigo derruba a sessão.
4. O interceptor de "401 → refresh" deve **ignorar respostas de `/auth/*`**. Um 401 ali é definitivo, e reagir com outro refresh cria laço.
5. Em 401 numa rota protegida: um refresh e **uma** repetição. Se falhar de novo, ou se o próprio refresh der 401, mande para o login.
6. 503 ou 504 no refresh (identity fora) **não** é sessão perdida.
7. **Repetir depois do refresh é seguro** quando o 401 veio da cadeia de filtros: ele chega antes de qualquer efeito. Um 401 repassado no checkout ou no `POST /orders` também chega antes de gravar o pedido, porque o catálogo e o orders validam o token antes de criar algo (`CartService.checkout`, `OrderService.createOrder`). A exceção são as mutações do carrinho: o cart grava e só depois consulta o catálogo para montar a resposta (`CartService.toResponse`). Um token que vence nesse intervalo devolve 401 com a mudança já feita. Depois de um 401 numa mutação do carrinho, releia `GET /cart` antes de repetir um `POST /cart/items` (inferido do código, sem teste).

### 4.6 Logout

- `POST /auth/logout` com `{ refreshToken }` → **204** sem corpo. Também dá 204 com token desconhecido, vazio ou já rotacionado (`RefreshTokenService.revoke`), então o front não detecta falha. Sem corpo, dá 400.
- Revoga só a família daquele login. Outros dispositivos continuam logados.
- O access token **continua válido até expirar** (até 15 min): apague-o localmente.

### 4.7 Papéis: quem pode o quê

| Ação | Anônimo | USER | ADMIN |
|---|---|---|---|
| cadastro, login, refresh, logout | sim | sim | sim |
| ver vitrine (`GET /products*`) | sim | sim | sim |
| criar, editar e apagar produto | 401 | 403 | sim |
| ler e alterar a própria conta | 401 | sim | sim |
| ler e alterar conta alheia (inclusive a senha, sem saber a antiga) | 401 | **403** | sim |
| carrinho e checkout (o próprio) | 401 | sim | sim (o próprio) |
| criar pedido; ver, confirmar e cancelar os próprios | 401 | sim | sim |
| ver, confirmar e cancelar pedido alheio | 401 | **404** | sim |
| listar pedidos | 401 | só os próprios | **todos** |
| `ship`, `deliver` | 401 | 403 (antes de procurar o pedido) | sim |
| ver o próprio pagamento | 401 | sim | sim |
| ver pagamento alheio | 401 | 404 | sim |
| `/admin/sales/**`, `/admin/notifications/**` | 401 | 403 | sim |
| listar usuários, trocar papel, apagar conta | não existe | não existe | não existe |

Regra de **papel** fica em `@PreAuthorize` nas controllers. Regra de **dono** fica na service (`OrderService.findOwnedOrThrow`, `PaymentService.requireOwnership`; `@PreAuthorize("#id.toString() == authentication.name or hasRole('ADMIN')")` em `UserController`).

### 4.8 Identity fora do ar e chave efêmera

- O Compose não define `JWT_PRIVATE_KEY`, então o identity gera uma chave RSA nova **a cada subida** (`JwtKeyConfig.java`):
  - access tokens emitidos antes dão **401 na hora** no próprio identity (`/users/**`);
  - os outros serviços guardam as chaves públicas em cache por 15 min, com tolerância a queda de mais 15 min (`JwkSetDecoderAutoConfiguration.java`), e podem continuar aceitando o token antigo por um tempo (inferido);
  - o **refresh continua valendo**, porque fica no Redis. Um refresh automático recupera a sessão.
- Com o identity parado, login, cadastro, refresh, `/users/**` e o import de contatos dão 503/504 pelo gateway. Access tokens válidos seguem aceitos em catálogo, carrinho, pedidos etc. (`e2e/.../IdentityE2E.java#shouldKeepAcceptingAValidTokenWhileTheIdentityIsDown`).

---

## 5. Endpoints por domínio

### 5.0 Mapa geral (pelo gateway)

| # | Método | Path | Papel | Sucesso | Erros principais |
|---|---|---|---|---|---|
| 1 | POST | `/auth/register` | público | 201 `UserResponse` + `Location` | 400, 409, 500 (senha > 72 bytes, provável) |
| 2 | POST | `/auth/login` | público | 200 `TokenResponse` | 400, 401 |
| 3 | POST | `/auth/refresh` | público | 200 `TokenResponse` | 400, 401 |
| 4 | POST | `/auth/logout` | público | 204 | 400 |
| 5 | GET | `/users/{id}` | dono ou ADMIN | 200 `UserResponse` | 400, 401, 403, 404 |
| 6 | PUT | `/users/{id}/name` | dono ou ADMIN | 200 `UserResponse` | 400, 401, 403, 404 |
| 7 | PUT | `/users/{id}/email` | dono ou ADMIN | 200 `UserResponse` | 400, 401, 403, 404, 409 |
| 8 | PUT | `/users/{id}/password` | dono ou ADMIN | 200 `UserResponse` | 400, 401, 403, 404, 500 (provável) |
| 9 | GET | `/products` | público | 200 `PageResponse<ProductResponse>` | 400 |
| 10 | GET | `/products/{id}` | público | 200 `ProductResponse` | 400, 404 |
| 11 | POST | `/products` | ADMIN | 201 `ProductResponse` + `Location` | 400, 401, 403 |
| 12 | PUT | `/products/{id}` | ADMIN | 200 `ProductResponse` | 400, 401, 403, 404 |
| 13 | DELETE | `/products/{id}` | ADMIN | 204 | 400, 401, 403, 404 |
| 14 | GET | `/cart` | autenticado | 200 `CartResponse` | 401, 404, 503 |
| 15 | POST | `/cart/items` | autenticado | 200 `CartResponse` | 400, 401, 404, 503 |
| 16 | PUT | `/cart/items/{productId}` | autenticado | 200 `CartResponse` | 400, 401, 404, 503, 500 (sem `quantity`) |
| 17 | DELETE | `/cart/items/{productId}` | autenticado | 200 `CartResponse` | 400, 401, 404, 503 |
| 18 | POST | `/cart/checkout` | autenticado | **200** `OrderResponse`, sem `Location` | 400, 401, 404, 409, 503, 504, 4xx repassado |
| 19 | GET | `/orders` | autenticado (ADMIN vê todos) | 200 `PageResponse<OrderResponse>` | 400, 401 |
| 20 | POST | `/orders` | autenticado | 201 `OrderResponse` + `Location` | 400, 401, 404, 503 |
| 21 | GET | `/orders/{id}` | dono ou ADMIN | 200 `OrderResponse` | 400, 401, 404 |
| 22 | POST | `/orders/{id}/confirm` | dono ou ADMIN | 200 `OrderResponse` | 400, 401, 404, 409 |
| 23 | POST | `/orders/{id}/cancel` | dono ou ADMIN | 200 `OrderResponse` | 400, 401, 404, 409 |
| 24 | POST | `/orders/{id}/ship` | ADMIN | 200 `OrderResponse` | 400, 401, 403, 404, 409 |
| 25 | POST | `/orders/{id}/deliver` | ADMIN | 200 `OrderResponse` | 400, 401, 403, 404, 409 |
| 26 | GET | `/payments/{id}` | dono ou ADMIN | 200 `PaymentResponse` | 400, 401, 404 |
| 27 | GET | `/payments/orders/{orderId}` | dono ou ADMIN | 200 `PaymentResponse` | 400, 401, 404 |
| 28 | GET | `/admin/sales/daily?from=&to=` | ADMIN | 200 `DailySalesResponse[]` | 400, 401, 403 |
| 29 | GET | `/admin/sales/top-products?limit=` | ADMIN | 200 `TopProductResponse[]` | 400, 401, 403 |
| 30 | POST | `/admin/sales/rebuild` | ADMIN | 200 `{ rebuilt }` | 401, 403, 503, 504 |
| 31 | GET | `/admin/notifications?state=` | ADMIN | 200 `PageResponse<NotificationResponse>` | 400, 401, 403 |
| 32 | POST | `/admin/notifications/contacts/import` | ADMIN | 200 `{ imported }` | 401, 403, 503, 504 |

Qualquer rota também pode dar 503 ou 504 do gateway. A captura do OpenAPI (§0) tem exatamente esses 27 paths e 32 operações; `DocumentationE2E.PUBLIC_ROUTES` confere os 27 paths (não os métodos).

**Ordem das checagens em rotas protegidas:** 401 (cadeia de filtros) → 400 de binding (UUID ou JSON inválido) → 403 (`@PreAuthorize`) → 404/400 de domínio → 409. Um USER que manda JSON malformado para uma rota de admin recebe **400, não 403**.

### 5.1 Identity

Fontes: `services/identity/.../user/*.java`, `UserIT`, `SecurityIT`, `RefreshTokenIT`.

#### `POST /auth/register` (público)

Entrada `UserDto`:

| Campo | Regras |
|---|---|
| `name` | obrigatório; não nulo, vazio nem só espaços (`isBlank`); até 150 (`String.length`). Não é aparado |
| `email` | obrigatório; até 150; regex `^[^@\s]+@[^@\s]+\.[^@\s]+$`; único (`uk_users_email`). Sem trim e sem lowercase no servidor |
| `password` | obrigatório; não vazio nem só espaços; 8 ou mais (`String.length`); sem regra de complexidade; acima de 72 bytes UTF-8 provavelmente dá 500 |

- Saída **201** `UserResponse { id, name, email, createdAt, updatedAt }`, com header `Location: http://<Host>/users/{id}`. Sem `role`, sem senha (`UserIT#shouldNeverExposeThePassword`). Use o `id` do corpo, não o `Location`.
- **Ordem das checagens** (`UserService.createUser`), uma por requisição:
  1. e-mail já cadastrado → **409** `Email <email> is already registered` (antes do formato);
  2. senha → 400 `errors.password`;
  3. hash BCrypt → 500 se passar de 72 bytes (provável);
  4. nome → 400 `errors.name`;
  5. formato ou tamanho do e-mail → 400 `errors.email`.
- Depois, de forma assíncrona: fato `user.registered`, e o notifications guarda o contato. **Não há e-mail de boas-vindas.**

Mensagens de campo (`User.java`):

| Chave | Mensagem |
|---|---|
| `name` | `User name cannot be empty` · `User name cannot exceed 150 characters` |
| `email` | `User email cannot be empty` · `User email cannot exceed 150 characters` · `User email is not a valid address` |
| `password` | `User password cannot be empty` · `User password must have at least 8 characters` |

#### `POST /auth/login`, `/auth/refresh`, `/auth/logout` (públicos)

| Rota | Entrada | Saída | Erros |
|---|---|---|---|
| login | `{ email, password }` (`LoginDto`, sem validação de formato) | 200 `TokenResponse` | 401 `Invalid credentials`; 400 `Invalid request body` |
| refresh | `{ refreshToken }` (`RefreshDto`) | 200 `TokenResponse` novo | 401 `Invalid refresh token`; 400 |
| logout | `{ refreshToken }` | 204 | 400 (corpo ausente ou JSON inválido) |

Exemplo de resposta de login e refresh:

```json
{ "token": "<JWT>", "type": "Bearer", "expiresIn": 900, "refreshToken": "<43 caracteres>" }
```

#### `/users/{id}` (dono ou ADMIN)

| Rota | Entrada | Ordem e erros | Observações |
|---|---|---|---|
| `GET /users/{id}` | — | 401 → 400 (UUID) → 403 → 404 | Para USER, conta alheia dá **403 mesmo se o id não existe** (o `@PreAuthorize` roda antes da busca). É 403 e não 404 de propósito: quem chama já conhece o id |
| `PUT /users/{id}/name` | `{ name }` | 404 antes da validação; 400 `errors.name` | mesmas regras do cadastro |
| `PUT /users/{id}/email` | `{ email }` | 404 → 409 → 400 `errors.email` | o **mesmo** e-mail (`equals` exato) é 200, sem fato. A troca é imediata, sem confirmação e sem pedir senha. O e-mail antigo fica livre na hora. O próximo login usa o novo. Os tokens atuais continuam valendo. Depois: fato `user.email-changed`, e os e-mails de pedido passam a ir para o endereço novo |
| `PUT /users/{id}/password` | `{ password }` (só a nova) | 404 → 400 `errors.password` → 500 (> 72 bytes, provável) (`UserService.updatePassword` busca antes de validar) | **não existe** "senha atual". Responde 200 `UserResponse` (não 204). **Não revoga** refresh nem access: outros dispositivos seguem logados. Uma senha recusada não altera o hash |

Todas devolvem `UserResponse`.

**Rotas que não existem:** `/users/me`, listagem ou busca de usuários, troca de papel, exclusão de conta, recuperação ou reset de senha, verificação de e-mail.

### 5.2 Catálogo

Fontes: `services/catalog/.../product/{ProductController,ProductService,Product,ProductDto,ProductResponse}.java`, `ProductIT`, `SecurityIT`.

`ProductResponse { id: uuid, name: string, description: string | null, price: number, createdAt, updatedAt }`.

#### `GET /products` (público)

- Query `page`, `size` (padrão 20, teto 100) e `sort` (padrão `createdAt,DESC`; §3.6). Ordem padrão: mais novo primeiro.
- 200 `PageResponse<ProductResponse>`. Erro: 400 `errors.sort`.
- **Não há busca, filtro, categoria, estoque, imagem, SKU nem flag ativo/inativo.** `ProductRepository` é um `JpaRepository` puro e `Product` tem seis campos.

#### `GET /products/{id}` (público)

200 `ProductResponse`; 404 `Could not find: <id>`; 400 se não for UUID.

#### `POST /products` (ADMIN)

Entrada `ProductDto { name, description, price }`. Só esses três campos são lidos. O `@Valid` do controller não tem efeito.

| Campo | Regras | `errors.<campo>` |
|---|---|---|
| `name` | obrigatório; não em branco; até 150. Não é aparado | `Product name cannot be empty` · `Product name cannot exceed 150 characters` |
| `description` | opcional (`null` ou `""`); até 1000 | `Product description cannot exceed 1000 characters` |
| `price` | obrigatório; maior que 0 | `Product price cannot be negative`, **a mesma mensagem para 0 e para `null`** |

- A validação para no primeiro erro, na ordem `name` → `description` → `price`.
- O backend **não** valida escala nem teto do preço (`DECIMAL(12,2)`, até 9.999.999.999,99).
- 201 com `Location: http://<Host>/products/{id}`. A resposta é montada do objeto em memória, sem reler do banco.

#### `PUT /products/{id}` (ADMIN)

- **Substituição completa:** os três campos são gravados, e `description` omitida vira `null`.
- 404 é checado antes da validação.
- Sem `@Version`: a última escrita vence.
- 200 `ProductResponse`.

#### `DELETE /products/{id}` (ADMIN)

- 204; 404. **Remoção física**, sem desfazer.
- Efeitos fora do catálogo:
  - nos carrinhos, o item some da leitura mas continua gravado e trava o checkout (§5.3);
  - pedidos antigos não mudam, porque copiaram nome e preço;
  - o `productId` de pedidos antigos passa a dar 404 no catálogo.

### 5.3 Carrinho

Fontes: `services/cart/.../cart/{CartController,CartService,Cart,CartItemDto,QuantityDto,CartResponse,CartItemResponse,CatalogService,OrdersService,OrdersClient}.java`, `CartIT`, `CheckoutIT`, `CartTest`.

**Modelo:**
- Um carrinho por usuário. O id é o `sub` do token: não há id na URL. Fica no Redis e guarda só `productId → quantity`, a `checkoutKey` e o TTL.
- **TTL de 7 dias** (604800 s), renovado a cada add, set e remove, inclusive `DELETE` de item ausente. `GET` não renova, e um checkout que falha também não.
- Nasce no primeiro `POST /cart/items`. Morre no checkout bem-sucedido ou no fim do TTL. **Não existe** rota para criar vazio nem `DELETE /cart`.
- Esvaziado item a item, o carrinho continua existindo: 200 com `items: []` e `total: 0`.
- **Não existe carrinho anônimo:** toda rota `/cart` exige token.

**Saída:**
- `CartResponse { id: string (= id do usuário), items: CartItemResponse[], total: number }`.
- `CartItemResponse { productId, productName, quantity, price, totalPrice }`. `price` é o **preço de hoje**, lido do catálogo. `totalPrice = price × quantity`.

**Como a resposta é montada** (`CartService.toResponse`):
- Uma chamada `GET /products/{id}` por item, **em sequência**.
- Produto que o catálogo responde 404 **some da resposta e do total**, mas **continua gravado** (o "item fantasma").
- Qualquer outro erro do catálogo derruba a requisição: 503, ou o 4xx repassado.
- A ordem dos itens **não é garantida** (`HashMap`).
- **Toda mutação grava antes de montar a resposta:** um 503 ou 4xx na montagem chega com a alteração **já feita**.

| Rota | Entrada | Comportamento | Erros (na ordem) |
|---|---|---|---|
| `GET /cart` | — | 200 `CartResponse` | 404 `Could not find: <userId>` (sem carrinho, ou TTL expirado); 503 `Service unavailable: catalog` (com itens e catálogo fora) |
| `POST /cart/items` | `{ productId: uuid, quantity: int }` | **soma** à quantidade (`Integer::sum`). Cria o carrinho no 1º item e responde 200 (não 201) | 404 `Could not find: <productId>` (o produto é checado **antes** da quantidade); 503; 400 `errors.quantity` `Quantity must be greater than zero`; 400 `Invalid request body` (inclusive `quantity: null`) |
| `PUT /cart/items/{productId}` | `{ quantity: int }` | **define** a quantidade; 0 remove; insere um produto que não estava, desde que o carrinho exista | `quantity` ausente ou `null` → **500** (unboxing de `Integer` nulo, antes de tudo); 404 sem carrinho; com `quantity > 0`, 404 de produto ou 503; 400 `errors.quantity` `Quantity cannot be negative` |
| `DELETE /cart/items/{productId}` | — | remove. Item ausente: 200 sem mudança. Funciona para produto já apagado do catálogo, sem consultá-lo | 404 sem carrinho; 503 (montagem da resposta com itens restantes) |
| `POST /cart/checkout` | sem corpo | ver abaixo | ver abaixo |

**Sem teto de quantidade nem estoque.** A soma não checa overflow.

#### `POST /cart/checkout`

Fluxo (`CartService.checkout`):
1. Prazo = agora + 4 s.
2. Sem carrinho → **404** `Could not find: <userId>`.
3. Sem itens **gravados** → **400** `items: Cannot checkout an empty cart` (`errors.items`).
4. Consulta cada item no catálogo:
   - os que voltam 404 geram **409** `Cart contains products that are no longer available: <id1>, <id2>`, com `errors: {}`, listando **todos** os ids. Nada é gravado e o carrinho fica intacto;
   - catálogo fora → 503 `Service unavailable: catalog`.
5. Prazo estourado aqui → 503 `Service unavailable: catalog`. O orders não é chamado.
6. Gera a `checkoutKey` (UUID) só se ainda não existe e grava, sem renovar o TTL.
7. `POST /orders` com `Idempotency-Key: <checkoutKey>`, o corpo `{ items: [{ productId, quantity }] }` e o token do usuário. O orders relê nome e preço no catálogo e os congela.
8. Prazo estourado nessa chamada → **503** `Service unavailable: orders`. Carrinho e chave são mantidos, mas **o pedido pode ter sido criado**.
9. Apaga o carrinho e responde **200** (não 201, sem `Location`) com o pedido **como o orders devolveu**. `status` é `PENDING` para pedido novo. Numa repetição com a mesma chave, vem o status **atual** do pedido existente.

O checkout **não confirma** o pedido. Para concluir a compra, chame `POST /orders/{id}/confirm`.

| Status | Quando | O pedido foi criado? |
|---|---|---|
| 200 | sucesso | sim |
| 400 | carrinho sem itens gravados | não |
| 401 | token ausente, ou recusado pelo catálogo ou pelo orders no meio | não |
| 404 | `Could not find: <userId>` (sem carrinho). Também 4xx repassado do orders, ex.: `Could not find: <productId>` | não |
| 409 | itens indisponíveis | não |
| 503 `Service unavailable: catalog` | catálogo fora, lento ou circuito aberto | **não** |
| 503 `Service unavailable: orders` | orders fora, lento, 5xx, circuito aberto ou prazo estourado | **talvez** |
| 504 (gateway) | o cart passou de 4,5 s | **talvez** |

**Idempotência:**
- Toda tentativa sobre o mesmo carrinho usa a mesma chave (`CartTest.shouldKeepTheSameCheckoutKeyAcrossAttempts`). Repetir depois de 503 `orders` ou de 504 devolve o mesmo pedido, sem duplicar (`CheckoutIT.shouldRetryWithTheSameKeyAfterTheOrderTakesLongerThanTheTimeout`; `CheckoutE2E`).
- **Nenhuma mutação apaga a chave.** Se o primeiro checkout criou o pedido e respondeu 503 ou 504, e o usuário edita o carrinho e repete, volta o pedido **antigo** e o carrinho editado é apagado (inferido).
- Dois checkouts **simultâneos** num carrinho ainda sem chave podem gerar chaves diferentes e **dois pedidos** (sem lock; inferido).

### 5.4 Pedidos

Fontes: `services/orders/.../order/{OrderController,OrderService,Order,OrderItem,OrderDto,OrderItemDto,OrderResponse,OrderItemResponse,OrderStatus}.java`, `OrderIT`, `OrderIdempotencyIT`, `SecurityIT`, `CatalogClientIT`, `OrderServiceTest`.

**Saída:**
- `OrderResponse { id, status, total, items: OrderItemResponse[], createdAt, updatedAt }`.
- `OrderItemResponse { id, productId, productName, unitPrice, quantity, subtotal }`.
- `productName` e `unitPrice` são **cópias congeladas** da compra. `subtotal = unitPrice × quantity`. `total` = soma dos `subtotal`.
- `items` vem completo também na listagem. A **ordem não é garantida** (sem `@OrderBy`): pode diferir entre o POST e o GET.
- `createdAt` e `updatedAt` truncados em microssegundos. `updatedAt` muda a cada transição.
- **Não vêm:** `userId`, `version`, data de cada status (`confirmedAt`...), status do pagamento, nome ou e-mail do cliente, a chave de idempotência, número amigável de pedido.

#### `GET /orders`

- Paginação: padrão 20, teto 100, `createdAt,DESC` (§3.6).
- USER recebe **só os próprios** (`findByUserId`). ADMIN recebe **todos** (`findAll`).
- **Sem filtro** por status, cliente ou data, e sem busca.
- Continua funcionando com o catálogo fora (`CatalogClientIT.shouldListTheCallerOrdersWhileTheCatalogIsDown`).

#### `GET /orders/{id}`

200; **404 se não existe ou não é seu** (ADMIN lê qualquer um); 400 se não for UUID.

#### `POST /orders` (criação direta, sem carrinho)

Entrada:

```json
{ "items": [ { "productId": "uuid", "quantity": 2 } ] }
```

Header opcional `Idempotency-Key` (§3.8).

| Campo | Regras | Erro |
|---|---|---|
| `items` | ao menos 1. Ausente, `null` ou `[]` → 400 sem chamar o catálogo | `errors.items` `Order must have at least one item` |
| `items[].productId` | UUID existente no catálogo | 404 `Could not find: <productId>`, com `errors: {}` |
| `items[].quantity` | inteiro maior que 0 | `errors.quantity`: `Quantity is required` / `Quantity must be greater than zero`. A chave **não** diz qual item |
| `Idempotency-Key` | até 64 caracteres | `errors.idempotencyKey` `Idempotency key must have at most 64 characters` |

- Nome, preço, subtotal, total e status são derivados. Se o cliente mandar algum, é ignorado.
- Itens com o mesmo `productId` não são agrupados. Não há limite de itens nem de quantidade.
- **Ordem** (`OrderService.createOrder`):
  1. chave já usada pelo usuário → devolve o pedido existente (201), **sem validar o corpo novo**;
  2. para cada item, em sequência: consulta o catálogo (404 ou 503), depois valida a `quantity`;
  3. lista vazia;
  4. tamanho da chave.
- Numa corrida com a mesma chave, quem perde recebe o pedido de quem ganhou.
- **201** com `Location: http://<Host>/orders/{id}`, `status: "PENDING"`.
- Erros adicionais: 503 `Service unavailable: catalog` (nada é criado); um 401 do catálogo é repassado; o gateway pode dar 503 ou 504. Depois de 503 ou 504, repita com a **mesma** chave. A repetição de uma chave já usada funciona mesmo com o catálogo fora.

#### Transições: `POST /orders/{id}/{confirm|cancel|ship|deliver}`

Sem corpo. Respondem 200 com o `OrderResponse` atualizado.

| Ação | De → para | Quem | 409 | Fato |
|---|---|---|---|---|
| `confirm` | `PENDING` → `CONFIRMED` | dono ou ADMIN (alheio → 404) | `Only pending orders can be confirmed` | `order.confirmed` |
| `cancel` | `PENDING` ou `CONFIRMED` → `CANCELLED` | dono ou ADMIN (alheio → 404) | `Only pending or confirmed orders can be cancelled` | `order.cancelled` |
| `ship` | `CONFIRMED` → `SHIPPED` | **só ADMIN** (USER → 403 antes de procurar) | `Only confirmed orders can be shipped` | `order.shipped` |
| `deliver` | `SHIPPED` → `DELIVERED` | **só ADMIN** | `Only shipped orders can be delivered` | `order.delivered` |

- Todas podem dar **409** `Resource was modified by another request, retry the operation` (concorrência, `@Version`).
- **Não são idempotentes:** o segundo clique dá 409.
- `ship` **não confere o pagamento** (`OrderService.shipOrder`).
- Cancelar um `CONFIRMED` **não estorna** (§5.5).

`GET /internal/orders/export` existe, mas **não passa pelo gateway** (404). É só para o sales.

### 5.5 Pagamento

Fontes: `services/payment/.../payment/{PaymentController,PaymentService,Payment,PaymentProcessor,PaymentResponse,PaymentStatus,PaymentGatewayClient,OrderConfirmedPaymentListener}.java`, `PaymentIT`, `PaymentControllerTest`.

Para o front, o serviço é **só leitura**. O pagamento nasce sozinho depois de `POST /orders/{id}/confirm`. **Não existe** rota para criar, pagar, aprovar, recusar, tentar de novo, estornar ou listar. O checkout **não** gera pagamento.

`PaymentResponse`:

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid | id do pagamento |
| `orderId` | uuid | pedido de origem |
| `amount` | number | `total` do pedido na confirmação, sempre maior que 0, BRL (sem campo de moeda) |
| `status` | `PENDING` \| `APPROVED` \| `DECLINED` \| `FAILED` | §6.2 |
| `gatewayReference` | string \| null | só em `APPROVED`. String opaca (local: `ch_` + 12 alfanuméricos) |
| `reason` | string \| null | só em `DECLINED` e `FAILED`, até 300 caracteres |
| `createdAt`, `updatedAt` | ISO-8601 UTC | `updatedAt` muda quando o pagamento é liquidado |

`userId` e a chave de idempotência **nunca** aparecem (`PaymentControllerTest.shouldNotExposeTheOwnerOrTheIdempotencyKey`).

| Rota | Uso | Erros |
|---|---|---|
| `GET /payments/orders/{orderId}` | **a que o front usa**: o pedido não traz o id do pagamento | 400 `errors.orderId`; 401; **404** `Could not find: <orderId>`; 503 (payment fora); 504 |
| `GET /payments/{id}` | só depois de obter o id pela rota acima | 400 `errors.id`; 401; 404 |

- Não há 403 aqui. Pagamento alheio dá 404, idêntico a "não existe".
- Sem token, um id malformado dá 401, porque o filtro vem antes do binding.
- `/payments/orders/undefined` dá 400 `errors.orderId`. `/payments/orders` (sem id) casa com `/payments/{id}` e dá 400 `errors.id`.

**O 404 de `/payments/orders/{orderId}` é ambíguo.** Cobre:
1. pedido ainda `PENDING`, que nunca terá pagamento enquanto não for confirmado;
2. confirmado, mas o fato ainda não chegou ao payment (janela normal, ou RabbitMQ ou orders fora);
3. pedido cancelado antes de confirmar;
4. pedido alheio ou inexistente.

Com o **próprio payment parado** a resposta é 503 do gateway, não 404. Quando ele volta, cobra, porque a fila é durável.

**Fluxo** (assíncrono em relação ao pedido):
1. `confirm` responde na hora e grava `order.confirmed` no outbox.
2. O relay publica (~1 s).
3. O payment grava `PENDING` com a chave `order-<orderId>`. Uma entrega repetida é ignorada: há no máximo um pagamento por pedido.
4. `POST /charges` no gateway externo, fora de transação.
5. Grava `APPROVED`, `DECLINED` ou `FAILED`.

O payment **não publica nada**: o status do pedido **não muda** com o resultado, e não há e-mail sobre pagamento.

Tempos esperados (derivados da configuração, não medidos):

| Cenário | O que `GET /payments/orders/{orderId}` mostra |
|---|---|
| caminho feliz (WireMock aprova) | 404 por ~1 s ou mais, `PENDING` breve, depois `APPROVED` |
| gateway 5xx ou conexão recusada | `PENDING` ~1,5 s, depois `FAILED` |
| gateway sem aceitar conexão (2 s) | `PENDING` até ~7,5 s, depois `FAILED` |
| gateway lento (3 s) | `PENDING` até ~10,5 s (no pior caso combinado, ~16,5 s), depois `FAILED` |
| circuito aberto (30 s) | `FAILED` quase imediato |
| payment cai no meio da cobrança | `PENDING` **para sempre** |

Os testes esperam 15 s (`PaymentIT.awaitPaymentStatus`, a cada 200 ms) e os e2e, até 60 s (`AbstractE2E.awaitApproval`).

### 5.6 Vendas (backoffice, ADMIN)

Fontes: `services/sales/.../sales/{SalesController,SalesService,SalesRebuildService,OrderViewProjector,SalesMessagingConfig,DailySalesResponse,TopProductResponse,RebuildResponse}.java`, `SalesReportIT`, `SalesRebuildIT`.

#### `GET /admin/sales/daily?from=YYYY-MM-DD&to=YYYY-MM-DD`

- `from` e `to` obrigatórios, `from ≤ to` e `ChronoUnit.DAYS.between(from, to) < 366` (até 366 dias contando os dois extremos).
- 200: array `[{ date: "YYYY-MM-DD", total: number }]`, **um item por dia** de `from` a `to`, em ordem crescente, com `0.00` nos dias sem venda.
- O dia é o da **confirmação** no fuso **America/Sao_Paulo**. Entram `CONFIRMED`, `SHIPPED` e `DELIVERED`; `CANCELLED` sai.
- Erros (400):
  - `errors.from`: `Parameter 'from' is required` · `Parameter 'from' has an invalid value` · `Start date must not be after end date`
  - `errors.to`: `Parameter 'to' is required` · `Parameter 'to' has an invalid value` · `Period cannot exceed 366 days`

#### `GET /admin/sales/top-products?limit=N`

- `limit` de 1 a 100, padrão 10. 400 `errors.limit`: `Limit must be between 1 and 100` ou `Parameter 'limit' has an invalid value`.
- 200: `[{ productId, productName, quantity: int64 }]`, ordenado por `quantity` decrescente e `productId` crescente. Pode vir `[]`.
- **Sem filtro de período** (desde sempre), conta **unidades, não receita**, exclui cancelados.
- `productName` é a cópia feita quando o pedido foi criado, não o nome atual do catálogo.

#### `POST /admin/sales/rebuild`

- Sem corpo. **Síncrono:** lê o export do orders em páginas de 100, com o token do admin.
- 200 `{ rebuilt: int }`, os pedidos **gravados**. `0` é sucesso normal.
- Não apaga nada antes. Os relatórios continuam respondendo durante a execução. Repetir é seguro.
- Erros:
  - 403 (o orders não é chamado);
  - 503 `Service unavailable: orders` (nada se perde);
  - 4xx do orders repassado;
  - **504 do gateway** depois de 4,5 s, enquanto o sales pode continuar trabalhando (cada página leva até 10 s).

**Consistência eventual dos relatórios:**
- Eles leem uma projeção em MongoDB alimentada pelos fatos de pedido, não o banco de pedidos.
- Um pedido recém-confirmado leva **alguns segundos** para aparecer.
- Um status que chega antes da confirmação espera 10 s, até 9 vezes (**~90 s**), e depois vai para a DLQ. Aí só um rebuild corrige. Na prática: um `order.cancelled` perdido mantém o pedido somado até o rebuild.
- Um cancelamento posterior **reduz o total de um dia passado**.
- Pedido `PENDING` nunca aparece.

### 5.7 Notificações (backoffice, ADMIN)

Fontes: `services/notifications/.../notification/{NotificationController,NotificationService,NotificationSender,Notification,NotificationMessages,NotificationResponse,NotificationState}.java`, `.../contact/{ContactImportController,ContactImportService,Contact,UserContactListener}.java`, ITs do serviço.

#### `GET /admin/notifications`

| Parâmetro | Padrão | Observação |
|---|---|---|
| `state` | `FAILED` | **um estado por vez**: `PENDING`, `SENT`, `NO_RECIPIENT`, `FAILED`, `SUPERSEDED`. Diferencia maiúsculas (`failed` → 400). Não há "todos" |
| `page`, `size`, `sort` | 0, 20, `updatedAt,DESC` | sem teto configurado (§3.6) |

- 200 `PageResponse<NotificationResponse>`, com `NotificationResponse { id, orderId, orderStatus: CONFIRMED|SHIPPED|DELIVERED|CANCELLED, state, attempts: int, lastError: string | null, updatedAt }`.
- **Não vêm:** e-mail, `userId`, `total`, `createdAt`, `nextAttemptAt`.
- Erros: 400 `errors.state`; 400 `errors.sort`; 401; 403.
- Para contadores por estado: 5 chamadas com `size=1`, lendo `totalElements`.
- **Não existem** reenvio, retry, detalhe por id nem filtro por pedido.

#### `POST /admin/notifications/contacts/import`

- Sem corpo. Síncrono: lê `/internal/users/export` do identity em páginas de 100, com o token do admin.
- 200 `{ imported: int }`, os e-mails novos ou trocados. `0` é sucesso.
- Só troca um e-mail por outro com instante mais novo.
- Erros:
  - 403 (o identity não é chamado);
  - 503 `Service unavailable: identity` (o que já foi copiado fica);
  - 4xx do identity repassado;
  - 504 do gateway (o serviço pode continuar).
- Rode **uma vez** ao começar a operar (§2.4). Só ajuda notificações ainda `PENDING`.

#### E-mails enviados (texto puro, remetente `MAIL_FROM`)

| `orderStatus` | Gatilho | Assunto | Corpo |
|---|---|---|---|
| `CONFIRMED` | `confirm` | `Pedido {orderId} confirmado` | `Seu pedido {orderId} foi confirmado. Total: R$ 1.234,56.` |
| `SHIPPED` | `ship` | `Pedido {orderId} enviado` | `Seu pedido {orderId} foi enviado.` |
| `DELIVERED` | `deliver` | `Pedido {orderId} entregue` | `Seu pedido {orderId} foi entregue.` |
| `CANCELLED` | `cancel` (inclusive de `PENDING`) | `Pedido {orderId} cancelado` | `Seu pedido {orderId} foi cancelado.` |

- `{orderId}` é o UUID completo. **Sem link** para o front.
- **Não há e-mail** no cadastro nem na criação do pedido.
- O destinatário é o e-mail **atual** do cliente, lido na hora do envio (`NotificationMessages.java`, `NotificationSender.java`).

### 5.8 Rotas próprias do gateway

| Rota | Observação |
|---|---|
| `GET /actuator/health` | público (`GatewayRoutingIT.shouldKeepItsOwnHealthOpen`). Reflete só o gateway. Corpo na captura de 2026-11-17: `{"groups":["liveness","readiness"],"status":"UP"}`, então existem `/actuator/health/liveness` e `/readiness`. Provavelmente sem CORS |
| `GET /swagger-ui.html` | UI única, com seletor dos 7 serviços. "Authorize" usa `bearerAuth` |
| `GET /v3/api-docs/swagger-config` | `urls: [{ name, url }]` dos 7 serviços |
| `GET /v3/api-docs/{serviço}` | OpenAPI 3.1.0 do serviço. `servers[0].url` = host do gateway (`DocumentationE2E.shouldPointTryItOutAtTheSingleEntry`) |

### 5.9 Tipos TypeScript de referência (escritos a partir do código)

```ts
type Uuid = string;
type IsoInstant = string;   // ISO-8601 UTC com 'Z', fração com nº variável de dígitos
type IsoDate = string;      // 'YYYY-MM-DD'
type Money = number;        // BigDecimal; BRL implícito

interface ErrorResponse { status: number; message: string; errors: Record<string, string>; }
interface PageResponse<T> { content: T[]; page: number; size: number; totalElements: number; totalPages: number; }

// identity
type Role = 'USER' | 'ADMIN';
interface JwtPayload { iss: 'orderflow'; sub: Uuid; roles: Role[]; iat: number; exp: number; }
interface TokenResponse { token: string; type: 'Bearer'; expiresIn: number; refreshToken: string; }
interface UserResponse { id: Uuid; name: string; email: string; createdAt: IsoInstant; updatedAt: IsoInstant; }
interface RegisterRequest { name: string; email: string; password: string; }
interface LoginRequest { email: string; password: string; }
interface RefreshRequest { refreshToken: string; }   // também é o corpo do logout

// catalog
interface ProductResponse { id: Uuid; name: string; description: string | null; price: Money; createdAt: IsoInstant; updatedAt: IsoInstant; }
interface ProductRequest { name: string; description?: string | null; price: number; }   // no PUT, mande os três

// cart
interface CartItemResponse { productId: Uuid; productName: string; quantity: number; price: Money; totalPrice: Money; }
interface CartResponse { id: string; items: CartItemResponse[]; total: Money; }   // id = sub do usuário
interface AddCartItemRequest { productId: Uuid; quantity: number; }   // > 0, soma
interface SetQuantityRequest { quantity: number; }                    // >= 0, sempre presente

// orders (também é o corpo 200 de POST /cart/checkout)
type OrderStatus = 'PENDING' | 'CONFIRMED' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';
interface OrderItemResponse { id: Uuid; productId: Uuid; productName: string; unitPrice: Money; quantity: number; subtotal: Money; }
interface OrderResponse { id: Uuid; status: OrderStatus; total: Money; items: OrderItemResponse[]; createdAt: IsoInstant; updatedAt: IsoInstant; }
interface CreateOrderRequest { items: { productId: Uuid; quantity: number }[]; }

// payment
type PaymentStatus = 'PENDING' | 'APPROVED' | 'DECLINED' | 'FAILED';
interface PaymentResponse { id: Uuid; orderId: Uuid; amount: Money; status: PaymentStatus; gatewayReference: string | null; reason: string | null; createdAt: IsoInstant; updatedAt: IsoInstant; }

// sales
interface DailySalesResponse { date: IsoDate; total: Money; }
interface TopProductResponse { productId: Uuid; productName: string; quantity: number; }
interface RebuildResponse { rebuilt: number; }

// notifications
type NotificationState = 'PENDING' | 'SENT' | 'NO_RECIPIENT' | 'FAILED' | 'SUPERSEDED';
interface NotificationResponse { id: Uuid; orderId: Uuid; orderStatus: Exclude<OrderStatus, 'PENDING'>; state: NotificationState; attempts: number; lastError: string | null; updatedAt: IsoInstant; }
interface ImportResponse { imported: number; }
```

---

## 6. Máquinas de estado

### 6.1 Pedido (`services/orders/.../order/Order.java`)

```
              confirm (dono/ADMIN)        ship (ADMIN)          deliver (ADMIN)
   PENDING ───────────────────► CONFIRMED ───────────► SHIPPED ───────────────► DELIVERED
      │                            │
      │ cancel (dono/ADMIN)        │ cancel (dono/ADMIN)
      └──────────────┬─────────────┘
                     ▼
                 CANCELLED

Terminais: DELIVERED, CANCELLED.
Nenhuma rota recebe o status alvo: o cliente pede a ação e o domínio decide.
```

| Status | Ações na UI (as demais dão 409) |
|---|---|
| `PENDING` | Confirmar, Cancelar (dono ou ADMIN) |
| `CONFIRMED` | Cancelar (dono ou ADMIN); Enviar (ADMIN) |
| `SHIPPED` | Entregar (ADMIN) |
| `DELIVERED`, `CANCELLED` | nenhuma |

Efeitos assíncronos: `confirm` dispara cobrança, vendas e e-mail. `ship`, `deliver` e `cancel` disparam vendas e e-mail.

### 6.2 Pagamento (`services/payment/.../payment/{Payment,PaymentProcessor}.java`)

```
 (sem pagamento: GET dá 404)
        │  order.confirmed consumido
        ▼
     PENDING ──┬──► APPROVED   gateway 2xx com status "approved" (sem diferenciar caixa)
               ├──► DECLINED   gateway 2xx com outro status, ou sem status
               └──► FAILED     timeout, conexão recusada, 5xx após 3 tentativas, 4xx do gateway,
                               circuito aberto, corpo ilegível, ou falha ao gravar o resultado

Terminais: APPROVED, DECLINED, FAILED (approve/decline/fail exigem PENDING).
```

| Status | `gatewayReference` | `reason` | Para a UI |
|---|---|---|---|
| `PENDING` | null | null | "processando". **Pode nunca terminar** se o serviço cair no meio (sem job de recuperação) |
| `APPROVED` | preenchido | null | pago |
| `DECLINED` | null | código do gateway (ex.: `insufficient_funds`) ou null | recusado. Traduza o código, com fallback genérico |
| `FAILED` | null | técnico: `<NomeDaExceção>: <mensagem>` | falha técnica. **Definitivo** (não há reprocessamento, apesar do texto do OpenAPI/README). **Não garante** que não houve cobrança. Não exiba `reason` |

O status do pagamento **não altera** o pedido. Um pedido `CONFIRMED`, `SHIPPED` ou `DELIVERED` pode ter pagamento `DECLINED` ou `FAILED`. Cancelar depois de `APPROVED` mantém `APPROVED`: **não há estorno** (o payment só escuta `order.confirmed`).

### 6.3 Notificação (`services/notifications/.../notification/Notification.java`)

```
 fato de pedido ──► PENDING (já vencida; uma por pedido e status: uk_notifications_order_status)
                      │  envio periódico a cada NOTIFICATIONS_POLL_INTERVAL (5 s), até 50 por vez
                      ├─ status mais novo do mesmo pedido já SENT ─► SUPERSEDED (não envia)
                      ├─ envio ok ──────────────────────────────────► SENT
                      └─ falha (servidor de e-mail, ou e-mail do cliente desconhecido)
                             attempts+1, lastError preenchido, nova tentativa em 30 s, 1, 2, 4 min
                             na 5ª falha: FAILED (falha de envio) ou NO_RECIPIENT (sem destinatário)

Terminais: SENT, FAILED, NO_RECIPIENT, SUPERSEDED. Da 1ª à última tentativa: ~7,5 min.
```

- `attempts` conta **só as falhas**: um `SENT` de primeira tem 0.
- O sucesso **não apaga** `lastError`: `SENT` ou `SUPERSEDED` podem trazer o erro de uma tentativa anterior. Use `state` para o selo.
- Em `NO_RECIPIENT`, `lastError` é sempre `No email known for the customer`.

### 6.4 Carrinho (para referência)

```
 (inexistente: GET 404) ──POST /cart/items──► ativo (TTL 7 d, renovado a cada mutação)
 ativo ──remover tudo──► ativo vazio (200, items: [], total 0; checkout dá 400)
 ativo ──checkout 200──► inexistente
 ativo ──TTL expira──► inexistente (checkoutKey perdida)
```

---

## 7. Jornadas ponta a ponta

Legenda: **[S]** síncrono, o resultado está na resposta. **[A]** assíncrono, vem depois por evento. **[P]** exige polling ou recarga. **[I]** estado intermediário que a UI precisa mostrar.

Fontes: `e2e/.../FullPurchaseE2E.java`, `AbstractE2E.java`, `CheckoutE2E.java`, `RoutesThroughGatewayE2E.java`, `IdentityE2E.java`, `NotificationsE2E.java`, `OrderFactsDeliveryE2E.java`, `SalesIsolationE2E.java`, `FailureIsolationE2E.java`, `CatalogOutageE2E.java`, `PaymentIsolationE2E.java`.

### 7.1 Cliente: cadastro e compra pelo carrinho

| # | Passo | Chamada | Tipo | O que a UI faz |
|---|---|---|---|---|
| 1 | cadastro | `POST /auth/register` | [S] 201, sem token | bloquear duplo envio (BCrypt leva centenas de ms; um segundo envio dá 409) |
| 2 | login | `POST /auth/login` | [S] 200 `TokenResponse` | decodificar `sub` e `roles`. [A] o notifications passa a conhecer o e-mail |
| 3 | vitrine | `GET /products`, `GET /products/{id}` | [S] | sem `Authorization` |
| 4 | montar carrinho | `POST /cart/items` (1º item), `PUT`/`DELETE /cart/items/{id}`, `GET /cart` | [S] | `GET /cart` 404 = carrinho vazio [I]. Preço é o **de hoje**. Ordenar itens no cliente. Depois de erro numa mutação, reler `GET /cart` |
| 5 | finalizar | `POST /cart/checkout` (sem corpo, sem `Idempotency-Key`) | [S] 200 `OrderResponse` `PENDING` | botão travado; spinner até ~5 s. 409 → limpar fantasmas e tentar de novo. 503 `orders` ou 504 → [I] "pode ter sido criado": repetir **sem editar** o carrinho, ou conferir `GET /orders`. Depois do sucesso, `GET /cart` dá 404 (normal) |
| 6 | confirmar a compra | `POST /orders/{id}/confirm` | [S] 200 `CONFIRMED` | passo explícito: o checkout **não** confirma. Não há formulário de cartão |
| 7 | pagamento | `GET /payments/orders/{orderId}` | [A] [P] | 404 ou `PENDING` = "processando pagamento" [I]. Parar em `APPROVED`, `DECLINED` ou `FAILED`. Teto de tempo (15–60 s) e depois "ainda processando, volte mais tarde". 503 = bloco de pagamento indisponível, não erro do pedido |
| 8 | e-mail "confirmado" | — | [A] | nada na UI. Chega no Mailpit em segundos |
| 9 | acompanhar | `GET /orders`, `GET /orders/{id}` | [S] na leitura, [P] para ver o que o admin fez | sem push: recarregar ou polling para ver `SHIPPED` e `DELIVERED`. Mostrar o status do pedido e o do pagamento lado a lado |

Resultado esperado no e2e: três e-mails (confirmado, enviado, entregue), pagamento `APPROVED` e o ranking atualizado, cada um em até 60 s.

### 7.2 Cliente: "comprar agora" (sem carrinho)

1. Gere `Idempotency-Key` (`crypto.randomUUID()`) para esta intenção de compra.
2. `POST /orders` `{ items: [...] }` com o header → [S] **201** `PENDING`. Em 503 ou 504, repita com a **mesma** chave. Se o usuário mudar os itens, gere chave nova.
3. Siga os passos 6 a 9 da §7.1.

### 7.3 Cliente: cancelar

- `POST /orders/{id}/cancel` → [S] 200 `CANCELLED`, a partir de `PENDING` ou `CONFIRMED`. Nos outros estados, 409.
- [A] e-mail "cancelado" (também para `PENDING`); o pedido sai dos relatórios.
- Pagamento: se já estava `APPROVED`, **continua `APPROVED`**. Não prometa reembolso. Se o pedido era `PENDING`, `GET /payments/orders/{id}` dá 404 para sempre.

### 7.4 Cliente: conta e sessão

| Ação | Chamada | Observação |
|---|---|---|
| ver perfil | `GET /users/{sub}` | `sub` do JWT |
| trocar nome | `PUT /users/{sub}/name` | [S] |
| trocar e-mail | `PUT /users/{sub}/email` | [S] imediato; o login passa a usar o novo; mesmo e-mail é 200. [A] e-mails seguintes vão para o novo endereço |
| trocar senha | `PUT /users/{sub}/password` | só a senha nova. Avisar que outros dispositivos continuam logados |
| renovar sessão | `POST /auth/refresh` | single-flight entre abas; trocar os dois tokens |
| sair | `POST /auth/logout` + apagar o access localmente | 204 sempre |

### 7.5 Admin: catálogo

1. Login → token com `roles: ["ADMIN"]`. Mostrar a UI de admin só quando `roles` contém `ADMIN`. O servidor aplica a regra de qualquer jeito.
2. `POST /products` → [S] 201. `PUT /products/{id}` (os três campos, pré-preenchidos) → [S] 200. `DELETE /products/{id}` → [S] 204, com diálogo de confirmação.
3. Efeitos: carrinhos passam a mostrar o preço novo na próxima leitura; um produto apagado vira item fantasma nos carrinhos. Pedidos antigos não mudam. Nada é assíncrono.

### 7.6 Admin: pedidos

1. `GET /orders` → **todos** os pedidos, sem filtro e sem dono (`OrderResponse` não tem `userId`). Só `page`, `size` e `sort` (ex.: `sort=status,asc`).
2. Antes de "Enviar", consultar `GET /payments/orders/{id}`: o backend **não impede** enviar com pagamento `DECLINED` ou `FAILED`. Qualquer bloqueio é só de UI.
3. `POST /orders/{id}/ship` → [S] 200 `SHIPPED`. `POST /orders/{id}/deliver` → [S] 200 `DELIVERED`. Em 409, recarregar o pedido.
4. [A] e-mails e projeção de vendas. Funcionam mesmo com payment, sales e notifications fora (`FailureIsolationE2E`).
5. O admin também pode `confirm` e `cancel` qualquer pedido. O dono do pagamento continua sendo o comprador.
6. Um painel "todos os pagamentos" exigiria `GET /orders` mais uma chamada por pedido, sem mostrar o dono.

### 7.7 Admin: vendas

1. `GET /admin/sales/daily?from&to` com datas do calendário de São Paulo (calcule "hoje" nesse fuso, não em UTC). `GET /admin/sales/top-products?limit`.
2. [A] [P] números eventuais: segundos, até ~90 s fora de ordem. Mostrar "atualizado há instantes" e botão de recarregar, sem cache longo.
3. `POST /admin/sales/rebuild` quando os números parecerem errados. 504 = "pode ainda estar rodando, tente de novo". `rebuilt: 0` é sucesso.

### 7.8 Admin: notificações

1. `GET /admin/notifications` (padrão `FAILED`) e `?state=NO_RECIPIENT`, para os clientes sem e-mail conhecido.
2. Uma vez, `POST /admin/notifications/contacts/import`.
3. Não há ação sobre uma notificação: só leitura. Mostrar o `orderId` inteiro com botão de copiar, para casar com o pedido e com o Mailpit.

### 7.9 Comportamento com serviços fora

| Fora | Continua | Falha |
|---|---|---|
| identity | access tokens já emitidos, em todas as rotas fora de `/users/**` | login, cadastro, refresh, `/users/**`, import de contatos (503/504) |
| catalog | `GET /orders`, `GET /orders/{id}`, carrinho sem itens | vitrine (503 do gateway); `GET /cart` com itens, mutações do carrinho, `POST /orders` e checkout (503 `Service unavailable: catalog`). Depois que volta, o circuito pode recusar por mais ~10 s |
| cart | pedidos | `/cart/**` (503) |
| orders | carrinho | checkout (503/504 em menos de 5 s, carrinho intacto; o retry cria **um** pedido), `/orders/**`, rebuild |
| payment | confirmar pedido | `GET /payments/**` (503). Quando volta, faz uma única cobrança |
| gateway de pagamento | tudo | cobranças viram `FAILED`. Com o circuito aberto, todas por 30 s |
| sales | pedidos | relatórios (503). Quando volta, os números se completam sem duplicar |
| notifications | pedidos | e-mails atrasam, depois saem |
| servidor de e-mail | tudo | e-mails atrasam (5 tentativas, ~7,5 min, depois `FAILED`) |

Regra para a UI: **degrade só o bloco afetado**, não a página inteira.

---

## 8. Validações para espelhar nos formulários

O servidor devolve **um erro por vez**, em inglês. Valide tudo no cliente e traduza pelo `status` e pela chave de `errors`. O `String.length` do JS conta em UTF-16, igual ao Java, então `.length` serve para os limites de caracteres.

| Formulário | Campo | Regra no cliente | Origem / chave de erro |
|---|---|---|---|
| cadastro, troca de nome | `name` | obrigatório, `trim()` não vazio, até 150 caracteres | `User.java` / `name` |
| cadastro, troca de e-mail | `email` | **aparar no cliente**; até 150; regex `^[^@\s]+@[^@\s]+\.[^@\s]+$` (mais estrita que `type=email`: `a@b` passa no HTML5 e leva 400) | `User.java` / `email`; duplicado → 409 |
| cadastro, troca de senha | `password` | não vazio nem só espaços; **8 caracteres no mínimo**; **72 bytes UTF-8 no máximo** (`new TextEncoder().encode(s).length <= 72`); sem regra de complexidade. Não há campo "senha atual" | `User.java` / `password`; acima de 72 bytes, provável 500 |
| login | `email`, `password` | obrigatórios (vazio dá 401, não 400); aparar o e-mail | `UserService.login` |
| produto | `name` | obrigatório, não em branco, até 150 | `Product.java` / `name` |
| produto | `description` | opcional, até 1000 | `description` |
| produto | `price` | número maior que 0, **até 2 casas**, menor que 10.000.000.000 (`DECIMAL(12,2)`; o backend não valida escala nem teto). Mensagem própria para 0 e vazio (o servidor diz `cannot be negative`) | `price` |
| carrinho, adicionar | `quantity` | inteiro ≥ 1. Teto de UX a definir pelo front (o backend não tem) | `Cart.java` / `quantity` |
| carrinho, definir | `quantity` | inteiro ≥ 0, **sempre presente** (ausente ou `null` → 500) | `quantity` |
| pedido direto | `items` | ao menos 1 | `Order.java` / `items` |
| pedido direto | `items[].quantity` | inteiro ≥ 1 (sem índice no erro) | `OrderItem.java` / `quantity` |
| pedido direto | `Idempotency-Key` | até 64 caracteres (`crypto.randomUUID()` tem 36) | `idempotencyKey` |
| vendas | `from`, `to` | `YYYY-MM-DD` montado como string de calendário (não `toISOString()`); `from ≤ to`; período de até 366 dias contando os extremos | `SalesService.java` / `from`, `to` |
| vendas | `limit` | inteiro de 1 a 100 | `limit` |
| notificações | `state` | um dos 5 valores, em MAIÚSCULAS | `state` |
| todas | ids no path | UUID válido antes de montar a URL (evita `/undefined`) | `id`, `orderId`, `productId` |
| listagens | `sort` | só os campos da §3.6 | `sort` |

---

## 9. Armadilhas para o frontend

Lista única, sem repetição, agrupada por tema. Os detalhes estão nas seções citadas.

### 9.1 Rede, CORS e headers
1. Origem exata: só `http://localhost:5173` por padrão. `127.0.0.1:5173`, a URL de rede do Vite (`--host`) e o `vite preview` (4173) levam 403 no preflight, com corpo texto `Invalid CORS request`. Ajuste `CORS_ALLOWED_ORIGINS` e recrie o gateway, ou use o proxy do Vite.
2. Só `Authorization`, `Content-Type` e `Idempotency-Key` passam no preflight. SDKs que injetam `traceparent`, `baggage`, `sentry-trace`, `X-Requested-With` ou `X-Request-Id` quebram todas as chamadas: desligue a propagação para a API.
3. Sem `PATCH` e sem `HEAD`. Sem cookies: nada de `credentials: 'include'` ou `withCredentials`.
4. Só `Location` é exposto. Navegue pelo `id` do corpo: o `Location` é absoluto, montado com o `Host` recebido, e atrás de proxy ou HTTPS pode apontar para outro host ou esquema.
5. Mande `Content-Type: application/json` em todo POST/PUT com corpo. `fetch` com string sem header manda `text/plain`, que provavelmente dá 500.
6. Respostas 204 (`DELETE /products/{id}`, `POST /auth/logout`) não têm corpo.

### 9.2 Autenticação e sessão
7. Não envie `Authorization` em `/auth/*`, `GET /products*`, health e docs. Um token expirado ali dá 401 (inclusive `Bearer undefined`), o refresh não é consumido e o logout não revoga nada. Cuidado com clientes gerados do OpenAPI, que declara `bearerAuth` global.
8. O access token está em `token`, não em `accessToken`. `expiresIn` é em segundos.
9. Cadastro não loga: chame o login em seguida. O login não devolve o usuário: decodifique o JWT (`sub`, `roles` é **array**). `UserResponse` não tem papel.
10. Refresh single-flight e coordenado entre abas. Troque os dois tokens a cada refresh. Nunca repita um refresh depois de 504, timeout ou erro de rede. O interceptor de 401 ignora `/auth/*`.
11. O 401 tem a mesma mensagem para token ausente, inválido ou expirado. Um 401 também pode vir **repassado** de outro serviço no meio de uma operação (checkout, `POST /orders`). Estratégia: um refresh e uma repetição. Nas mutações do carrinho, releia `GET /cart` antes de repetir (§4.5, item 7).
12. Reiniciar o identity (Compose sem `JWT_PRIVATE_KEY`) invalida os access tokens: espere 401 em massa, que o refresh resolve.
13. Logout responde 204 sempre e não invalida o access token: apague-o localmente.
14. Troca de senha não pede a atual e não desloga outras sessões. Troca de e-mail é imediata.

### 9.3 Erros
15. `errors` sempre é objeto (`{}` quando vazio) e tem no máximo uma chave. Decida pela combinação de `status` e chave de `errors`. Não mostre `message` crua: é inglês técnico, com prefixo `"campo: "`.
16. A ordem das checagens decide qual erro aparece: no cadastro, o 409 de e-mail duplicado vem antes da senha. Em rota de admin, um JSON malformado dá 400 antes do 403. Num item inexistente, o 404 vem antes da validação de `quantity`.
17. 404 em pedido ou pagamento alheio = "não encontrado". Já `/users/{id}` alheio dá **403**, e `ship`/`deliver` dão 403 para USER antes de procurar o pedido.
18. Há dois 409 nas transições: estado inválido (`Only ... can be ...`) e concorrência (`Resource was modified...`). Nos dois, recarregue o pedido.
19. 503 `Service unavailable` (sem sufixo) vem do gateway: o próprio serviço está fora. 503 `Service unavailable: <nome>` é uma dependência interna fora. Com circuito aberto, o 503 sai na hora durante 10 s (30 s no pagamento).
20. 504 **não** significa que a operação falhou (checkout, `POST /orders`, rebuild, import).
21. Método, `Content-Type` ou subpath errados dentro de um prefixo roteado provavelmente dão 500, não 405/415/404.

### 9.4 Paginação e formatos
22. `page` começa em 0. `size` da resposta é o tamanho aplicado (teto 100 silencioso em produtos e pedidos), não a contagem de itens. Não há `hasNext`. Mandar `sort` substitui o padrão. Campo de sort desconhecido dá 400.
23. Dinheiro é `number`. `JSON.parse` perde zeros à direita (`100.00` vira `100`). Formate com `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })` e mostre os totais do servidor, sem somar floats. Se somar dias do relatório, arredonde para centavos.
24. `Instant` com número variável de casas: use parser ISO completo, não compare strings, converta para o fuso local. `date` do relatório diário é data de calendário: `new Date('2026-10-10')` vira 09/10 em BRT.
25. Campos desconhecidos na entrada são ignorados em silêncio: um typo vira "campo ausente".

### 9.5 Catálogo
26. Não há busca, filtro, categoria, estoque nem imagem. Busca no cliente só cobre a página carregada (até 100 itens).
27. `PUT /products/{id}` substitui tudo: `description` omitida é apagada. Envie os três campos.
28. `DELETE /products/{id}` é físico e cria itens fantasma nos carrinhos. Peça confirmação.
29. A resposta de `POST/PUT /products` ecoa o preço como enviado. O próximo GET mostra o valor do banco.

### 9.6 Carrinho e checkout
30. `GET /cart` 404 = carrinho vazio (nunca criado, finalizado ou expirado), não erro. Compare o id da `message` com o `sub` ou com o `productId` para distinguir "sem carrinho" de "produto inexistente".
31. `POST /cart/items` **soma** e não é idempotente: clique duplo dobra. Para seletor de quantidade e botões +/−, use `PUT`. O `PUT` exige carrinho existente: o primeiro item vai por `POST`.
32. Toda mutação grava antes de responder: depois de qualquer erro numa mutação, releia `GET /cart` antes de repetir.
33. **Item fantasma:** produto apagado some de `GET /cart` e do total, mas trava o checkout com 409. Os ids estão só na `message`: extraia os UUIDs por regex, `DELETE /cart/items/{id}` em cada um, avise o usuário e finalize de novo. Um carrinho que aparece vazio pode dar 409, não 400.
34. Ordem dos itens instável: ordene no cliente.
35. Preço vivo no carrinho, congelado no pedido. Mostre o `total` devolvido pelo checkout e avise se diferir.
36. Checkout responde **200** com `OrderResponse` (não 201, sem `Location`). O status nem sempre é `PENDING` numa repetição.
37. Depois de 503 `orders` ou 504 no checkout: **não deixe editar o carrinho** antes de repetir. Desabilite "finalizar" enquanto houver requisição em voo (dois checkouts simultâneos podem criar dois pedidos).
38. O carrinho visitante, se existir, é só do front: depois do login, um `POST /cart/items` por produto, **somando** ao carrinho do servidor.
39. Carrinhos grandes ficam lentos (uma consulta ao catálogo por item, até 1,5 s cada) e podem bater 504. Evite rajadas de chamadas ao carrinho.

### 9.7 Pedidos
40. `Idempotency-Key` em `POST /orders`: um UUID por intenção de compra, reaproveitado só nas repetições. Nunca vazio, nunca fixo. Repetição com outro corpo devolve o pedido antigo, sem aviso.
41. O checkout não confirma: a compra só "acontece" com `POST /orders/{id}/confirm`.
42. Transições não são idempotentes: trave o botão e, em 409/503/504, recarregue `GET /orders/{id}`.
43. `OrderResponse` não tem `userId`, datas por status nem status de pagamento. Uma timeline só tem `createdAt` e `updatedAt`.
44. O 404 de produto apagado no `POST /orders` tem `errors: {}`, com o UUID só na `message`. O 400 de `quantity` não diz qual item.
45. `productId` de pedido antigo pode apontar para produto apagado: não monte link obrigatório para o produto.

### 9.8 Pagamento
46. Polling com teto: 404 e `PENDING` = "processando". Ambos podem durar para sempre. Pare em estado terminal.
47. O 404 também significa "nunca haverá" (pedido não confirmado ou cancelado antes) e "não é seu". Só consulte pagamento de pedidos que passaram por `CONFIRMED`; para `CANCELLED`, consulte e trate 404 como "sem pagamento".
48. Não há botão de pagar, tentar de novo ou estornar. `FAILED` é definitivo e não garante "não cobrado". Cancelar não estorna.
49. `reason` de `FAILED` é técnico (pode conter nome de exceção e URL interna): não exiba. `reason` de `DECLINED` é código: traduza, com fallback.
50. Localmente todo pagamento sai `APPROVED`: use mocks para as outras telas.

### 9.9 Backoffice
51. Todas as rotas `/admin/**` exigem ADMIN. Esconda o menu pela claim, mas trate 401/403 em toda chamada: a claim reflete o papel na emissão.
52. `top-products` não tem período, conta unidades e usa o nome da época da compra. Não reuse o filtro de datas do relatório diário.
53. Relatórios mudam para trás: cancelamentos tiram vendas de dias passados.
54. Rebuild e import são síncronos e longos: trave o botão, trate 504 como "tente de novo" e `0` como sucesso. Um 4xx neles pode vir do orders ou do identity.
55. Notificações: `state` em maiúsculas, um estado por vez, padrão `FAILED`. `lastError` preenchido não indica erro atual. Paginar `PENDING` enquanto o envio roda pode pular ou repetir itens (`updatedAt` muda).

### 9.10 OpenAPI e codegen
56. Não confie cegamente nos tipos gerados: sem `required`/`nullable`, `*/*`, checkout como `JsonNode`, `pageable` objeto em notifications, `bearerAuth` global, faltam 401/403/503 (§3.10). Use a §5.9.

---

## 10. O que o backend NÃO oferece hoje

O planejamento do front precisa contar com estas lacunas (ou com trabalho no backend):

**Conta e identidade**
- `GET /users/me`. A identidade vem do JWT.
- Listagem e busca de usuários; troca de papel; promoção a admin; exclusão de conta.
- Recuperação ou reset de senha ("esqueci minha senha"); verificação de e-mail; confirmação de troca de e-mail; senha atual na troca de senha.
- Revogar todas as sessões; listar dispositivos logados.
- Login social, MFA, cookies de sessão.

**Catálogo**
- Busca textual, filtros (nome, preço, faixa), categorias.
- Imagem ou upload de arquivo: nenhuma rota recebe arquivo, nem multipart.
- Estoque, SKU, ativo/inativo, soft delete, `PATCH`, controle de versão na edição.

**Carrinho**
- Carrinho anônimo, `DELETE /cart` ("esvaziar"), limite de quantidade, aviso de mudança de preço ou de produto removido.

**Pedidos**
- Endereço de entrega, frete, cupom ou desconto (o `OrderDto` só tem `items`).
- Número amigável do pedido; `userId` e dados do cliente na resposta; filtros por status, cliente ou data; busca.
- Datas por status (timeline); edição de pedido.

**Pagamento**
- Rota de pagamento ou formulário de cartão; escolha de método; nova tentativa; estorno ou reembolso; listagem de pagamentos; status de pagamento dentro do pedido; impedir envio sem pagamento aprovado.

**Notificações**
- Notificações para o cliente na API (só e-mail); preferências de notificação; reenvio, detalhe ou filtro por pedido no admin; e-mail de boas-vindas ou de pedido criado.

**Transversal**
- Push (WebSocket, SSE): tudo o que é assíncrono exige polling.
- Rotas agregadas (pedido + pagamento + cliente).
- i18n das mensagens de erro (tudo em inglês).
- Campo de moeda (BRL implícito).
- Rate limiting, `ETag`, cache HTTP, id de rastro na resposta.
- Endpoint de saúde agregada dos serviços (o health do gateway só fala dele mesmo).

---

## 11. Incertezas que sobraram

Nada abaixo tem teste no repositório nem foi observado ao vivo. Confirme na stack rodando antes de depender disso.

**Verificação ao vivo**
1. O OpenAPI e o health não foram relidos ao vivo (stack parada na redação e na revisão de 2026-11-18). A captura de 2026-11-17 15:19 confirma `servers[0].url`, `*/*`, a ausência de `required`/`nullable`, `pageable` como objeto obrigatório em notifications, o schema `JsonNode` do checkout e o corpo do `/actuator/health`. Releia ao vivo antes de gerar cliente, caso o código tenha mudado depois do commit de referência.
2. Serialização real: `null` explícito ou ausente em `gatewayReference`, `reason` e `lastError`; `100.00` ou `100`; número de casas nas datas, que pode diferir entre a resposta de uma escrita e a leitura seguinte.

**Autenticação**
3. 401 com Bearer inválido em rota pública: esperado pelo Spring Security 7.1.1 (sem `BearerTokenResolver` customizado em `SecurityConfig.java`), sem teste.
4. Dois `/auth/refresh` simultâneos com o mesmo token: dois tokens válidos ou família derrubada.
5. Depois de reiniciar o identity com chave nova, por quanto tempo catalog e orders aceitam tokens antigos (cache JWKS de 15 min mais tolerância a queda).
6. Relógio do navegador contra o do identity: o `exp` vem do relógio do servidor, e um relógio local adiantado ou atrasado distorce a renovação por `exp`. Contar `expiresIn` a partir do recebimento evita a dependência. (A folga do servidor, 60 s, já foi conferida: §4.4.)
7. Falha ao buscar o JWKS com cache vazio pode virar 500 fora do formato `ErrorResponse`.
8. Ausência de `WWW-Authenticate` no 401.

**Validação e erros**
9. Senha acima de 72 bytes dá 500 no cadastro e na troca (deduzido do bytecode do BCrypt na spring-security-crypto 7.1.1). Login com senha longa só falha com 401.
10. Caixa do e-mail: a collation não está fixada em lugar nenhum. A imagem é `mysql:8.4` (`docker-compose.yml`), e nem a migração (`services/identity/src/main/resources/db/migration/V1__create_users_table.sql`) nem `docker/mysql/init` definem `COLLATE` ou `CHARACTER SET`. Vale o padrão do servidor (provável `utf8mb4_0900_ai_ci`, que ignora caixa e acento). Se for assim, a unicidade e o login ignoram caixa, e trocar o próprio e-mail mudando só a caixa daria 409 (`updateEmail` usa `equals`).
11. 405, 415 e subpath inexistente dentro de um prefixo roteado saem como 500 (handler genérico).
12. `productId` nulo em `POST /cart/items` ou `POST /orders`: provável 500, contando falha no circuit breaker do catálogo (cinco requisições assim poderiam abrir o circuito para todos). `{"items":[null]}` provavelmente dá 500.
13. `quantity` **ausente** em `POST /cart/items` (campo `int` de record): vira 0 (400 de domínio, ou 404 antes se o produto não existe) ou falha na desserialização (400 `Invalid request body`). `quantity: null` dá 400 `Invalid request body`, por `FAIL_ON_NULL_FOR_PRIMITIVES`, sem teste.
14. `quantity` decimal provavelmente truncado (`ACCEPT_FLOAT_AS_INT`): `0.5` vira 0 e dá 400.
15. Preço com mais de 2 casas ou acima de `DECIMAL(12,2)`, e subtotal estourando `DECIMAL(12,2)`: arredondamento pelo MySQL ou 500.
16. `Idempotency-Key: ""` vira uma chave fixa válida, e não "sem chave". Não foi verificado se o gateway repassa header vazio.
17. Texto exato do 404 do gateway (`Could not find: /<path>`): o teste só confere o status.
18. Ordenação por propriedade aninhada (`sort=items.productName`) pode duplicar pedidos na página.

**CORS e gateway**
19. Requisição real (não preflight) de origem desconhecida: provável 403 `Invalid CORS request`.
20. Se as respostas de erro geradas pelo próprio gateway (404, 503, 504) levam `Access-Control-Allow-Origin`. Se não levarem, o navegador mostra erro de rede em vez do status.
21. `Location` atrás do proxy do Vite, que depende de `changeOrigin`.
22. Leitura cross-origin de `/actuator/health`, provavelmente bloqueada.

**Assíncrono e tempos**
23. Os tempos do pagamento (1,5 s, 7,5 s, 10,5 s, até ~16,5 s até `FAILED`) foram derivados da configuração e da ordem padrão Retry/CircuitBreaker do resilience4j, não medidos. Também não se sabe quantos pagamentos abrem o circuito.
24. Reentrega do listener do payment com o banco fora (container factory padrão, sem DLQ). O desfecho conhecido pelo código é o pagamento preso em `PENDING`.
25. Rebuild e import continuam no serviço depois do 504 do gateway, e quanto tempo levam com o volume atual.
26. Teto de página em `/admin/notifications` (padrão do Boot, 2000).
27. O 400 de `sort` desconhecido só está testado no catálogo. Em orders e notifications, presume-se o mesmo handler.

**Carrinho e pedidos**
28. Mutações simultâneas do mesmo carrinho podem perder atualizações (sem versão nem lock).
29. Checkout simultâneo num carrinho sem chave pode criar dois pedidos. Editar o carrinho depois de um 503/504 e repetir devolve o pedido antigo e apaga o carrinho editado.
30. A ordem efetiva dos itens do pedido no MySQL não foi observada.
31. Chave do Redis `cart:<userId>` (convenção do Spring Data Redis, não observada).

**Outros**
32. Vocabulário de `reason` em `DECLINED`: o único exemplo no código é `insufficient_funds`. Não há contrato fechado.
33. A regra de que ADMIN vê pagamentos alheios vem só do código: nenhum teste do payment usa token ADMIN.
34. `from` e `to` não ISO podem ser aceitos conforme a locale do servidor. Use só `YYYY-MM-DD`.
35. Se a API do Mailpit aceita CORS do navegador.
