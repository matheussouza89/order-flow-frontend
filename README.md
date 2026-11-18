# OrderFlow — frontend

SPA do OrderFlow: a loja (vitrine, carrinho, pedidos, conta) e o backoffice
(catálogo, pedidos, vendas, notificações). Consome a API do backend
(repositório `orderFlow`) pelo gateway em `http://localhost:8080`.

Projeto de portfólio. Como no backend, cada decisão fica registrada abaixo com
o porquê. Tudo o que o front precisa saber da API está em
[`docs/backend-reference.md`](docs/backend-reference.md), citada aqui como §N.

**Estado:** planejamento. Stack, design system, identidade visual e estrutura
decididos; CI e deploy em aberto.

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
  Condensed 700, cobalto. No carrinho vira miniatura de 44px, com grade de 8px
  e só a inicial.
- **Uma ousadia só:** o quadro é o único elemento chamativo; o resto da tela
  fica quieto.
- **Forma:** cantos de 4px nos controles e de 6px nas caixas, e bordas de 1px
  no lugar de sombra.
- **Backoffice:** a mesma base, com barra lateral azul-marinho e tabelas
  densas. O botão de ação de cada linha segue a máquina de estados (§6.1):
  Enviar só em `CONFIRMED`, Entregar só em `SHIPPED`.

Tokens:

| Token | Valor | Uso |
|---|---|---|
| `bg` | `#F2F5F9` | fundo da página |
| `surface` | `#FFFFFF` | caixas e tabelas |
| `ink` | `#10213F` | texto |
| `muted` | `#4A5A78` | texto secundário |
| `line` | `#C9D3E3` | bordas |
| `bar` | `#10213F` | barra do topo e lateral do backoffice |
| `accent` | `#2350D8` | botão principal, links, quadro do produto |
| `accent-soft` | `#E6EDF9` | fundo do quadro, item ativo |
| `warning-*` | fundo `#FBF3E4`, borda `#E9D3A8`, texto `#6B4A10`, marcador `#B7791F` | avisos (item que saiu do catálogo) |
| `danger-*` | texto `#B42B2B`, borda `#E7C2C2` | botão de contorno para ação destrutiva (Cancelar pedido) |

Tipografia, uma família só com três papéis:

- **IBM Plex Sans:** texto e interface, base de 14px.
- **IBM Plex Sans Condensed 700:** títulos, preços e totais, com algarismos
  tabulares.
- **IBM Plex Mono:** só para IDs, como o UUID do pedido com botão Copiar.

Status: quadrado colorido mais rótulo em português, nunca só a cor. Quadrado
vazado indica que algo ainda está pendente. O pulso do pagamento em
processamento para com `prefers-reduced-motion`.

| Status | Rótulo | Cor | Marcador |
|---|---|---|---|
| Pedido `PENDING` | Pendente | `#7A88A3` | vazado |
| Pedido `CONFIRMED` | Confirmado | `#2350D8` | cheio |
| Pedido `SHIPPED` | Enviado | `#7A4DD8` | cheio |
| Pedido `DELIVERED` | Entregue | `#2F855A` | cheio |
| Pedido `CANCELLED` | Cancelado | `#A0AEC4` | cheio |
| Pagamento `PENDING` | Processando | `#B7791F` | vazado, pulsando |
| Pagamento `APPROVED` | Aprovado | `#2F855A` | cheio |
| Pagamento `DECLINED` | Recusado | `#C53030` | cheio |
| Pagamento `FAILED` | Falhou | `#8B2525` | cheio |

O detalhe do pedido mostra as etapas Pendente, Confirmado, Enviado e Entregue.

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

### ESLint e Prettier, não Biome

O Biome é mais rápido e é uma ferramenta só, mas os plugins de que o projeto
depende só existem para ESLint: fronteiras entre features, `react-hooks`,
TanStack Query e TanStack Router.

### pnpm

O `node_modules` estrito impede usar uma dependência que não está no
`package.json`. Também é mais rápido e economiza disco.

## Em aberto

- Modo escuro e as telas restantes do backoffice (produtos, vendas,
  notificações), incluindo as cores dos status de notificação.
- CI e deploy.
- Decisões de produto que a API deixa para o front, como o carrinho de
  visitante (§9.6, item 38) e os tempos de polling.
