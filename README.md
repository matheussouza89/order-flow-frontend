# OrderFlow — frontend

SPA do OrderFlow: a loja (vitrine, carrinho, pedidos, conta) e o backoffice
(catálogo, pedidos, vendas, notificações). Consome a API do backend
(repositório `orderFlow`) pelo gateway em `http://localhost:8080`.

Projeto de portfólio. Como no backend, cada decisão fica registrada abaixo com
o porquê. Tudo o que o front precisa saber da API está em
[`docs/backend-reference.md`](docs/backend-reference.md), citada aqui como §N.

**Estado:** planejamento. Stack, design system, identidade visual, estrutura,
decisões de produto e telas definidos; estados de tela, CI e deploy em aberto.

## Stack

| Camada | Escolha |
|---|---|
| Framework | React + TypeScript, build com Vite (SPA) |
| Rotas | TanStack Router, rotas por arquivo |
| Dados da API | TanStack Query |
| HTTP | axios |
| Formulários e validação | React Hook Form + Zod |
| Componentes | shadcn/ui (Radix) + Tailwind |
| Tabela, datas, gráficos, avisos | TanStack Table, react-day-picker, Recharts, Sonner |
| Documentação dos componentes | Storybook |
| Mocks | MSW |
| Testes | Vitest, Testing Library, Playwright |
| Lint e formatação | ESLint + Prettier |
| Pacotes | pnpm |

## Decisões

### SPA com Vite, não Next.js

Nada aqui precisa de renderização no servidor: não há SEO a ganhar, e toda tela
depende do usuário logado ou do admin. O gateway já libera CORS para
`http://localhost:5173`, a porta padrão do Vite (§3.2).

Alternativas consideradas:

- **Next.js com BFF:** o servidor do Next guardaria os tokens em cookie
  httpOnly, e o navegador nunca veria o JWT. É o único ganho real. O custo é um
  segundo servidor entre o navegador e o gateway, o refresh rotativo
  reimplementado nele e o CORS a mudar (porta 3000).
- **Angular:** traz rotas, HTTP, interceptors e formulários prontos, e a DI
  lembra o Spring. Ficou atrás pelo ecossistema do React, que tem peças feitas
  para os problemas desta API (próxima decisão), e pelo mercado.
- **Vue e Svelte:** mercado menor no Brasil, sem vantagem específica aqui.

### Dados da API no TanStack Query

A API tem mais estado assíncrono do que formulário:

- o pagamento nasce depois da confirmação e exige polling com teto, parando em
  `APPROVED`, `DECLINED` ou `FAILED` (§5.5);
- os relatórios de vendas são eventuais (§5.6);
- várias chamadas **não podem** ser repetidas sozinhas: refresh, transições de
  pedido, `POST /cart/items` (§3.8).

O TanStack Query resolve os três com `refetchInterval` condicional, `retry`
por consulta e invalidação depois de mutações. SWR é fraco em mutações; RTK
Query exigiria Redux.

Não há biblioteca de estado global: o estado do servidor vive no Query, e a
sessão num módulo próprio.

### Rotas tipadas com TanStack Router

Paginação, ordenação e filtros moram na URL (`page`, `size`, `sort`, `state`).
Com o TanStack Router, esses parâmetros são tipados e validados por schema: um
`sort` fora da lista da §3.6 ou um `state` em minúsculas é corrigido na URL, em
vez de virar 400 na API. Os loaders das rotas se integram ao Query para
pré-carregar dados.

Rotas por arquivo: a árvore é gerada, e cada arquivo de rota só junta as
features.

Alternativa: **React Router**, mais conhecido e mais simples, mas com parâmetros
de URL sem tipo.

### axios como cliente HTTP

Os interceptors de requisição e resposta são o lugar natural para anexar o
token e reagir ao 401, num padrão que a maioria dos devs React já conhece.

O custo aceito: a fila de refresh com interceptors é fácil de errar. Por isso
as regras da §4.5 ficam explícitas no interceptor:

- um refresh por vez (promise compartilhada), coordenado entre abas;
- 401 vindo de `/auth/*` é definitivo, nunca dispara refresh;
- em 401 de rota protegida, um refresh e **uma** repetição;
- refresh nunca é repetido depois de timeout, 504 ou erro de rede;
- nada de `Authorization` nas rotas públicas (§4.1), nada de `withCredentials`,
  e nenhum retry automático (sem `axios-retry`).

Alternativas: **fetch com wrapper próprio** (zero dependências, controle total,
mais código) e **ky** (wrapper pequeno do fetch, mas o refresh fica escondido
em hooks).

### Tokens: access em memória, refresh no localStorage

O backend entrega os dois tokens no corpo, sem cookie (§4.2). Onde guardar é
decisão do front.

- **Access em memória:** nunca é gravado; some no reload e é renovado pelo
  refresh.
- **Refresh no localStorage:** a sessão sobrevive ao reload e é a mesma em
  todas as abas.

O refresh é rotativo, e reusar um token já usado derruba o login inteiro
(§4.5). Duas abas renovando juntas deslogam o usuário. A coordenação usa Web
Locks: quem pega a trava relê o refresh do storage antes de renovar; se outra
aba já trocou, usa o novo em vez de rotacionar de novo.

Descartadas:

- **sessionStorage:** duplicar a aba copia o refresh. As duas abas renovam com
  o mesmo token, e a sessão cai.
- **Tudo em memória:** login a cada reload ou aba nova.
- **Tudo no localStorage:** expõe também o access, sem ganho.

Risco aceito: um XSS consegue ler o refresh, como em qualquer SPA sem cookie.

### Formulários com React Hook Form e Zod

O servidor devolve um erro por vez, em inglês (§3.4). O cliente valida tudo
antes e mostra todos os erros juntos, em português.

Os schemas Zod espelham as regras da §8 (regex de e-mail, senha de 8
caracteres a 72 bytes, preço com até 2 casas). Os mesmos schemas validam as
respostas da API nas bordas, porque o OpenAPI não marca `required` nem
`nullable` (§3.10). Os tipos vêm da §5.9, escritos à mão, não de codegen.

Erros do servidor são traduzidos pela combinação de `status` e chave de
`errors`, nunca pela `message`.

### Design system sobre shadcn/ui e Tailwind

O shadcn/ui copia o código dos componentes para o repositório. O design system
é nosso: tokens em variáveis CSS, variantes e componentes que podem ser lidos e
alterados. As primitivas Radix trazem teclado e acessibilidade.

Complementos para o backoffice: TanStack Table (listagens paginadas), Calendar
com react-day-picker (período do relatório diário), Recharts (gráfico de
vendas) e Sonner (avisos).

Alternativas:

- **Mantine:** biblioteca completa e backoffice rápido, mas o design system
  seria a configuração de um tema.
- **MUI:** visual Material difícil de escapar, pesado, e o DataGrid avançado é
  pago.
- **Do zero (Radix + CSS):** máximo aprendizado, semanas antes da primeira tela.

Risco: shadcn sem personalização tem cara de "mais um site shadcn". A
identidade visual (próxima decisão) existe para resolver isso.

### Identidade visual: ficha técnica, paleta Prancheta

A loja vende periféricos, como os dados de teste do backend. A API não tem
imagem de produto (§5.2, §9.5), então o card não pode depender de foto. Por
isso cada produto é apresentado como uma ficha técnica sobre papel
quadriculado de prancheta.

- **Quadro do produto:** fica no lugar da foto. Fundo `accent-soft`, grade
  cobalto (`#2350D826`) a cada 12px e a primeira palavra do nome em Plex Sans
  Condensed 700, cobalto. No carrinho vira miniatura de 48px, com grade de 8px
  e só a inicial.
- **Card do produto:** quadro, nome (link para o produto), `description` em até
  duas linhas, preço e "Adicionar ao carrinho". A descrição é texto livre e
  pode vir `null` (§5.2); o espaço fica reservado, e os cards de uma linha
  mantêm a mesma altura.
- **Uma ousadia só:** o quadro é o único elemento chamativo; o resto da tela
  fica quieto. Por isso o botão do card é de contorno: seis botões cheios
  disputariam com o quadro.
- **Forma:** cantos de 4px nos controles e de 6px nas caixas, e bordas de 1px
  no lugar de sombra.
- **Backoffice:** a mesma base, com barra lateral azul-marinho e tabelas
  densas. O botão de ação de cada linha segue a máquina de estados (§6.1):
  "Marcar enviado" só em `CONFIRMED`, "Marcar entregue" só em `SHIPPED`.

Tokens:

| Token | Valor | Uso |
|---|---|---|
| `bg` | `#F2F5F9` | fundo da página |
| `surface` | `#FFFFFF` | caixas e tabelas |
| `ink` | `#10213F` | texto |
| `muted` | `#4A5A78` | texto secundário |
| `line` | `#C9D3E3` | bordas |
| `line-soft` | `#E3E9F2` | divisórias entre linhas de lista e tabela |
| `bar` | `#10213F` | barra do topo e lateral do backoffice |
| `ring-dark` | `#6F93FF` | foco e item ativo sobre a barra azul-marinho |
| `accent` | `#2350D8` | botão principal, links, quadro do produto |
| `accent-soft` | `#E6EDF9` | fundo do quadro, item ativo |
| `warning-*` | fundo `#FBF3E4`, borda `#E9D3A8`, texto `#6B4A10`, marcador `#B7791F` | avisos (item que saiu do catálogo) |
| `danger-*` | texto `#B42B2B`, borda `#E7C2C2` | botão de contorno para ação destrutiva (Cancelar pedido) |

Tipografia, uma família só com três papéis:

- **IBM Plex Sans:** texto e interface, base de 16px.
- **IBM Plex Sans Condensed 700:** títulos, preços e totais, com algarismos
  tabulares.
- **IBM Plex Mono:** só para IDs, como o UUID do pedido.

Tamanhos:

- Texto que se lê: 16px. Descrição, preço unitário e datas: 14px. Rótulos e
  cabeçalhos de tabela: 13px, o mínimo.
- Título da tela 28px, preço no card 22px, total do carrinho 24px.
- Alvo de toque de 44px na loja. Os botões das linhas do backoffice têm 36px,
  porque a tabela precisa ser densa.
- Espaçamento na escala 4, 8, 12, 16, 24 e 32px.

Botões:

- Um único botão cheio por tela, na ação principal (Finalizar compra).
- O resto é de contorno. A ação destrutiva usa o mesmo contorno, em vermelho.
- Rótulo com verbo e objeto: "Adicionar ao carrinho", "Marcar enviado",
  "Copiar número". Botões repetidos numa lista levam o item no nome acessível
  ("Adicionar Mouse sem fio ao carrinho").
- Quantidade, ordenação e paginação são controles de verdade: o − fica
  desativado na quantidade 1, a ordenação tem rótulo visível e só oferece
  campos aceitos pela API (§3.6), e "Anterior" fica desativado na primeira
  página.

Foco e movimento:

- Foco: anel de 2px em `accent`, afastado 2px, sem animação. Sobre a barra
  azul-marinho, em `ring-dark`.
- Hover: o botão clareia e sobe 1px; o card com link sobe 2px; o clique afunda
  1px. Duração de 150ms na entrada, 120ms na saída e 90ms no clique.
- Nada se move com a rolagem. Com `prefers-reduced-motion`, só as cores mudam.

Status: quadrado colorido mais rótulo em português, nunca só a cor. Quadrado
vazado indica que algo ainda está pendente. Enquanto o polling do pagamento
roda, o marcador de "Processando" gira. No teto do polling, ele para, e o
bloco mostra "Sem resposta ainda" com o botão "Verificar de novo", porque
`PENDING` pode não terminar (§6.2).

| Status | Rótulo | Cor | Marcador |
|---|---|---|---|
| Pedido `PENDING` | Pendente | `#7A88A3` | vazado |
| Pedido `CONFIRMED` | Confirmado | `#2350D8` | cheio |
| Pedido `SHIPPED` | Enviado | `#0891B2` | cheio |
| Pedido `DELIVERED` | Entregue | `#2F855A` | cheio |
| Pedido `CANCELLED` | Cancelado | `#A0AEC4` | cheio |
| Pagamento `PENDING` | Processando | `#B7791F` | vazado; gira durante o polling |
| Pagamento `APPROVED` | Aprovado | `#2F855A` | cheio |
| Pagamento `DECLINED` | Recusado | `#C53030` | cheio |
| Pagamento `FAILED` | Falhou | `#8B2525` | cheio |
| Notificação `PENDING` | Na fila | `#7A88A3` | vazado |
| Notificação `SENT` | Enviada | `#2F855A` | cheio |
| Notificação `FAILED` | Falhou | `#8B2525` | cheio |
| Notificação `NO_RECIPIENT` | Sem destinatário | `#B7791F` | cheio |
| Notificação `SUPERSEDED` | Substituída | `#A0AEC4` | cheio |

O detalhe do pedido mostra as etapas Pendente, Confirmado, Enviado e Entregue,
com marcadores quadrados. O título usa o código curto (os 8 primeiros
caracteres do UUID), o mesmo da lista do backoffice, para que cliente e admin
falem do mesmo pedido. O UUID completo fica abaixo, com "Copiar número".

No carrinho, cada linha tem "Remover" (`DELETE /cart/items/{id}`). Em larguras
médias, quantidade e subtotal descem para baixo do nome.

Direções descartadas:

- **Tecla:** cada produto vira uma tecla, e os botões afundam ao clicar. Fala a
  língua de quem compra periférico, mas a metáfora não serve ao backoffice.
- **Caixa:** cada produto ganha uma cor de embalagem e o nome em letra enorme.
  É viva, mas a cor sai do id do produto e exige paleta fechada com contraste
  conferido.

Paletas descartadas, dentro da ficha técnica:

- **Placa:** verde de placa de circuito com cobre. É mais ligada ao assunto,
  mas o verde disputa com o verde de "aprovado" e de "entregue".
- **Milimetrado:** grafite sobre papel milimetrado, com marca-texto amarelo. É
  a mais neutra, mas a mais sem cara.

Revisão com a [taste-skill](https://github.com/Leonxlnx/taste-skill), um guia
de design para agentes. Ela foi feita para sites institucionais, então só
entrou o que vale para telas de aplicativo:

- **Adotado:** nada de roxo ("Enviado" era `#7A4DD8`; o verde-azulado testado
  em seguida ficava igual ao verde de "Entregue"), texto e alvos de toque
  maiores, foco visível, resposta a hover e clique, rótulos visíveis, botões
  com verbo e objeto, e nenhum ponto piscando.
- **Recusado:** fotos de banco de imagens no lugar do quadro. A API não tem
  campo de imagem, então não há como escolher a foto certa para cada produto
  que o admin cadastra.
- **Fora do escopo:** hero, seções do tamanho de uma tela, animações de
  rolagem e efeitos.

### Storybook como vitrine do design system

Cada componente aparece em todos os estados: carregando, vazio, erro e os
pagamentos `DECLINED`, `FAILED` e `PENDING` sem fim, que o backend local nunca
produz (o WireMock aprova tudo, §2.5). As histórias usam os mesmos handlers do
MSW.

Custo aceito: setup e manutenção a mais.

### Mocks com MSW

Os mesmos handlers servem ao desenvolvimento, aos testes e ao Storybook. Cobrem
o que a stack local não reproduz: pagamento recusado ou com falha, 404 de
pagamento, 503 de serviço fora, 504 do gateway, item fantasma no checkout.

### Testes: Vitest, Testing Library e Playwright

- **Vitest:** usa a mesma configuração do Vite.
- **Testing Library:** testa o comportamento visível, não a implementação.
- **Playwright:** jornadas ponta a ponta contra a stack real do Compose. A API
  do Mailpit é chamada pelo runner, não pela SPA (§2.5).

### Organização por feature, como no backend

```
src/
├── app/            providers, router, query client
├── routes/         rotas por arquivo, finas
│   ├── (loja)/     vitrine, carrinho, pedidos, conta
│   └── admin/      carregadas sob demanda, só para ADMIN
├── features/
│   ├── auth/       sessão, refresh, guarda de papel
│   ├── catalog/
│   ├── cart/
│   ├── orders/
│   ├── payments/
│   ├── account/
│   ├── sales/            backoffice
│   └── notifications/    backoffice
├── shared/
│   ├── api/        instância do axios, interceptors, ErrorResponse
│   ├── ui/         componentes shadcn e tokens
│   └── lib/        dinheiro em BRL, datas no fuso de São Paulo, UUID
└── mocks/          handlers do MSW
```

Cada feature tem `api.ts`, `schemas.ts`, `components/`, `hooks/` e um
`index.ts`, que é a única porta de entrada. Uma regra de ESLint proíbe importar
o interior de outra feature: é o equivalente ao package-private do backend, com
o lint no lugar do compilador.

Alternativas:

- **Por tipo** (`components/`, `hooks/`, `services/`): recusada pelo mesmo
  motivo do backend, porque mexer numa feature obriga a abrir quatro pastas.
- **Feature-Sliced Design:** camadas e regras verificadas por ferramenta, mas
  com cerimônia demais para este tamanho; a fronteira entre feature, entity e
  widget vira dúvida constante.

### Um app só, com o backoffice carregado sob demanda

Loja e backoffice compartilham sessão, design system e cliente HTTP. As rotas
`/admin` são carregadas sob demanda: o código só é baixado por quem tem
`roles: ["ADMIN"]`. A guarda de rota esconde o menu, mas a autoridade é o
servidor: 401 e 403 são tratados em toda chamada (§9.9).

Alternativa: **dois apps num monorepo**, com separação forte, mas com
workspace, pacote de UI compartilhado e dois deploys. Mais infraestrutura que
produto.

### Compra em dois passos: finalizar e confirmar

O checkout cria o pedido como `PENDING` e não confirma; a cobrança só começa
com `POST /orders/{id}/confirm` (§5.3, §7.1).

- "Finalizar compra" chama `POST /cart/checkout` e abre o pedido para revisão,
  com o total já congelado.
- A revisão tem "Confirmar compra" e "Cancelar pedido". Se o total congelado
  for diferente do que o carrinho mostrava, a revisão avisa (§9.6, item 35).
- Depois de confirmar, a tela acompanha o pagamento (decisão de polling, mais
  abaixo).

Por quê: segue o domínio, e o cliente vê uma mudança de preço antes de pagar.
Se o confirm falhar, o que sobra é um pedido `PENDING` claro, com o botão de
confirmar ainda lá.

Alternativas:

- **Confirmar automático depois do checkout:** um clique só, mas o cliente não
  veria a mudança de preço, e uma falha entre as duas chamadas criaria um
  estado difícil de explicar.
- **Automático quando o total bate:** um clique no caso comum, mas são dois
  caminhos para testar e explicar.

Custo aceito: um clique a mais, e pedidos abandonados ficam `PENDING`.

Fora desta versão: o "comprar agora" sem carrinho (§7.2). Só o carrinho
compra, então há um único caminho para desenhar e testar. O custo é que a
`Idempotency-Key` gerada no cliente fica sem demonstração; o checkout do
carrinho já mostra a repetição segura depois de 503 e 504.

### Login antes do carrinho

A vitrine é pública. "Adicionar ao carrinho" sem sessão leva ao login ou ao
cadastro e, na volta, adiciona o item: a intenção sobrevive ao login.

Por quê: a API não tem carrinho anônimo, e toda rota `/cart` exige token
(§5.3). Com um único carrinho, o do servidor, não há o que sincronizar.

Alternativa: **carrinho local com junção no login**, um `POST /cart/items` por
produto (§9.6, item 38). A soma duplicaria o que já estava no servidor, os
preços guardados envelheceriam, um produto apagado daria 404 na junção, e
seriam dois carrinhos para manter.

Custo aceito: atrito antes da primeira compra.

### Cancelamento só antes de confirmar

O cliente cancela apenas pedidos `PENDING`. Depois de confirmado, só o admin
cancela.

Por quê: cancelar não estorna (§7.3, §9.8 item 48). Um cliente que cancelasse
um pedido pago ficaria sem o produto e sem o dinheiro.

Alternativa: **permitir também em `CONFIRMED`, com um aviso** de que o valor
não volta sozinho. Segue a API, mas deixa o cliente fazer algo que o
prejudica.

Custo aceito: o front restringe o que a API permite, e o cliente não tem
saída pela loja depois de confirmar.

### Polling do pagamento e atualização do pedido

Não há push: tudo o que é assíncrono chega por polling ou releitura (§1.2).

- **Pagamento:** `GET /payments/orders/{orderId}` a cada 1 s nos primeiros
  15 s, depois a cada 3 s até 60 s. Para em `APPROVED`, `DECLINED` ou
  `FAILED`. No teto, para e mostra "Sem resposta ainda" com "Verificar de
  novo", que recomeça o ciclo. 404 conta como "processando" só para pedidos
  que passaram por `CONFIRMED` (§9.8, item 47); 503 deixa indisponível só o
  bloco de pagamento.
- **Pedido:** relido ao abrir a tela e quando a janela volta ao foco, sem
  polling contínuo.

Por quê: no caminho feliz o resultado sai em 1 a 2 s; com falha no gateway,
em até cerca de 16,5 s; os e2e do backend esperam 60 s (§5.5). São cerca de
30 chamadas no máximo. Já enviado e entregue são ações do admin, que numa loja
real levam horas ou dias; na demo, mudar o pedido no admin em outra aba e
voltar já mostra o status novo.

Alternativas:

- **2 s fixo, teto de 30 s:** mais simples, mas o caminho feliz demora mais
  para aparecer, e o teto fica abaixo do dos e2e.
- **1 s fixo, teto de 60 s:** até 60 chamadas, o dobro.
- **Polling de 30 s no pedido:** atualiza sozinho, mas faz chamadas constantes
  por algo raro.

### Envio só com pagamento aprovado

No backoffice, "Marcar enviado" só fica ativo quando o pagamento do pedido
está `APPROVED`. A lista consulta o pagamento dos pedidos `CONFIRMED` da
página.

Por quê: o backend não confere o pagamento antes do envio (§5.4, §7.6), então
daria para enviar um pedido não pago sem perceber.

Alternativas:

- **Avisar e deixar enviar:** o admin decide casos especiais, mas depende de
  ler o aviso.
- **Não consultar:** zero chamadas extras, sem proteção nenhuma.

Custo aceito: uma chamada de pagamento por pedido confirmado na página (até 20
no tamanho padrão). A regra vive só no front: a API continua aceitando o envio
de qualquer cliente HTTP.

### ESLint e Prettier, não Biome

O Biome é mais rápido e é uma ferramenta só, mas os plugins de que o projeto
depende só existem para ESLint: fronteiras entre features, `react-hooks`,
TanStack Query e TanStack Router.

### pnpm

O `node_modules` estrito impede usar uma dependência que não está no
`package.json`. Também é mais rápido e economiza disco.

## Telas

Desenhadas em mockups (fora do repositório) com a identidade e as decisões
acima. Em todas, o pedido aparece pelo código curto (os 8 primeiros caracteres
do UUID), e onde o número inteiro importa há "Copiar número".

Loja:

- **Catálogo:** grade de cards, ordenação com rótulo e paginação. Sem busca,
  porque a API não tem.
- **Detalhe do produto:** quadro grande, descrição inteira, quantidade e
  "Adicionar ao carrinho", o botão cheio da tela. Depois do clique, a
  confirmação aparece ao lado, com "Ver carrinho".
- **Entrar e Criar conta:**
  - Quem chega por "Adicionar ao carrinho" vê um aviso do que vai acontecer na
    volta.
  - O cadastro mostra todos os erros juntos, em português, e entra na conta
    logo em seguida, porque a API não loga no cadastro (§9.2, item 9).
  - E-mail ou senha errados dão um aviso só, sem dizer qual dos dois.
- **Carrinho:** aviso de item que saiu do catálogo, quantidade, "Remover" e
  "Finalizar compra".
- **Revisão do pedido:** pedido `PENDING` com aviso de preço alterado e o valor
  antigo riscado, "Nada foi cobrado ainda", "Confirmar compra" e "Cancelar
  pedido".
- **Pedido:**
  - Etapas, itens com o preço congelado e o bloco de pagamento.
  - O bloco cobre processando, aprovado, recusado (código do gateway
    traduzido), falhou (sem prometer que não houve cobrança) e indisponível
    (503 só no bloco).
  - Depois de confirmado, não há botão de cancelar, e uma linha explica por
    quê.
- **Meus pedidos:** código, data, itens resumidos, status e total; a linha
  inteira abre o pedido. O pagamento não aparece aqui, porque custaria uma
  chamada por pedido. Sem pedidos, a tela convida a ver o catálogo.
- **Conta:** um formulário por dado (nome, e-mail, senha nova), cada um com seu
  botão. Os textos avisam que o e-mail troca na hora e que trocar a senha não
  desconecta outros aparelhos (§9.2, item 14).

Backoffice:

- **Pedidos:** status, pagamento e total. "Marcar enviado" só fica ativo com o
  pagamento aprovado; "Marcar entregue" aparece nos enviados.
- **Produtos:** lista com nome, preço e data da última alteração, "Novo
  produto", "Editar" e "Excluir".
- **Editar produto:**
  - Os três campos vão juntos, porque o `PUT` substitui tudo (§9.5, item 27).
  - A descrição tem contador até 1000 caracteres.
  - O preço aceita no máximo duas casas e tem mensagem própria para zero e
    vazio.
- **Excluir produto:** diálogo que diz o efeito invisível ao admin (o produto
  sai dos carrinhos) e abre com o foco em "Manter produto".
- **Vendas:**
  - Período de até 366 dias, com datas no fuso de São Paulo.
  - Total, dias com venda, melhor dia e um gráfico com uma barra por dia,
    inclusive os dias sem venda.
  - "Mais vendidos" conta unidades desde o início, como a API entrega.
  - "Lido às" e "Atualizar" para a consistência eventual; "Refazer relatório"
    fica à parte.
- **Notificações:**
  - Um botão por estado, com a contagem de cada um (cinco chamadas com
    `size=1`). Abre em "Falhou".
  - A tabela mostra pedido, aviso, falhas, último erro e data. A tela só lê,
    porque não existe reenvio.
  - "Importar contatos" vem com a explicação de quando usar.

## Em aberto

- Estados de cada tela além do vazio de "Meus pedidos": carregando, erro e
  serviço fora do ar.
- Modo escuro, e a conferência do layout no celular.
- CI e deploy.
