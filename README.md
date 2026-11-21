# OrderFlow — frontend

SPA do OrderFlow: a loja (vitrine, carrinho, pedidos, conta) e o backoffice
(catálogo, pedidos, vendas, notificações). Consome a API do backend
(repositório `orderFlow`) pelo gateway em `http://localhost:8080`.

Projeto de portfólio. Como no backend, cada decisão fica registrada abaixo com
o porquê. Tudo o que o front precisa saber da API está em
[`docs/backend-reference.md`](docs/backend-reference.md), citada aqui como §N.

**Estado:** planejamento. Stack, design system, identidade visual, estrutura,
decisões de produto, telas, estados de tela, modo escuro e celular definidos;
CI e deploy em aberto.

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
  no lugar de sombra. Só o que flutua sobre a tela tem sombra (ver "Camadas e
  avisos").
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
| `on-accent` | `#FFFFFF` | texto do botão cheio |
| `accent-soft` | `#E6EDF9` | fundo do quadro, item ativo |
| `raised` | `#FFFFFF` | diálogo, menu e calendário (no escuro, mais claro que `surface`) |
| `info-*` | fundo `#E6EDF9`, borda `#C3D2F0`, texto `#1B3A8F`, marcador `#2350D8` | aviso neutro (sessão encerrada, pedido confirmado depois de demora) |
| `success-*` | fundo `#EAF5EE`, borda `#B9DCC6`, texto `#1F5A3A`, marcador `#2F855A` | toast e confirmação na tela (Adicionado ao carrinho) |
| `warning-*` | fundo `#FBF3E4`, borda `#E9D3A8`, texto `#6B4A10`, marcador `#B7791F` | avisos (item que saiu do catálogo) |
| `danger-*` | fundo `#FBEFEF`, borda `#E7C2C2`, texto `#7A1F1F`, marcador `#B42B2B` | aviso de erro; o marcador também é o texto do botão destrutivo (Cancelar pedido) e do erro de campo |
| `disabled` | texto `#6B7A96` sobre `bg`, borda `line` | controle desativado |
| `shadow-float` | `0 12px 32px #10213F33` | diálogo, menu, calendário e toast |
| `overlay` | `#10213F80` | fundo escurecido atrás do diálogo |

Tipografia, uma família só com três papéis:

- **IBM Plex Sans:** texto e interface, base de 16px.
- **IBM Plex Sans Condensed 700:** títulos, preços e totais, com algarismos
  tabulares.
- **IBM Plex Mono:** só para IDs, como o UUID do pedido.

Tamanhos:

- Texto que se lê: 16px. Descrição, preço unitário, datas e rótulos de campo:
  14px. Cabeçalhos de tabela, eixos do gráfico e contadores: 13px, o mínimo.
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

- Foco: anel de 2px em `accent`, afastado 2px, sem animação, igual em botões,
  links e campos. Sobre a barra azul-marinho, em `ring-dark`. O halo
  translúcido do shadcn foi descartado: tem contraste de cerca de 1,4:1, abaixo
  dos 3:1 que o foco precisa.
- Hover: o botão clareia e sobe 1px; o card com link sobe 2px; o clique afunda
  1px. Duração de 150ms na entrada, 120ms na saída e 90ms no clique.
- Nada se move com a rolagem. Com `prefers-reduced-motion`, só as cores mudam.

Campos:

- Rótulo acima, em 14px e peso 500. Dica abaixo, em 14px e `muted`, ligada ao
  campo por `aria-describedby`.
- Altura igual à do botão ao lado: 44px na loja e nos formulários, 36px nos
  filtros do backoffice.
- Erro abaixo do campo, em `danger`, com um quadrado de 8px na frente; a borda
  do campo fica vermelha (`aria-invalid`). O texto diz como corrigir ("Escreva
  o e-mail completo, como nome@exemplo.com").
- Contador (descrição do produto) em 13px, alinhado à direita.

Camadas e avisos:

- Diálogo, menu, calendário e toast ficam por cima da página e são os únicos
  com sombra (`shadow-float`), sem perder a borda de 1px. Sem sombra, um menu
  branco sobre uma caixa branca se confunde com o que está embaixo. O diálogo
  escurece o fundo com `overlay`.
- Toast só confirma ação que deu certo: "Número copiado", "Nome salvo",
  "Produto salvo". Usa a família `success`, aparece embaixo à direita (embaixo
  e no centro no celular) e some em 4 segundos.
- Erro nunca vai para toast. Fica na tela, perto de onde aconteceu (no campo ou
  no bloco do botão), porque um aviso que some sozinho pode não ser lido.
- Confirmação com próximo passo também fica na tela, como "Adicionado ao
  carrinho" com "Ver carrinho".

Ícones:

- Lucide, que o shadcn já usa, com traço de 2px na cor do texto: 16px ao lado
  de texto, 20px sozinho num botão.
- Só onde ajudam a reconhecer: fechar o diálogo, copiar, carrinho no topo,
  Menu do celular, botão de tema, setas do calendário e seta do select. Ícone
  sem texto tem nome acessível.
- Status continua sendo o quadrado. Títulos, cards e o menu lateral não levam
  ícone: ícone em tudo dá cara de template.

Gráfico de vendas:

- Uma barra por dia, toda em `accent`, com cantos retos como os marcadores. O
  dia sem venda tem um traço de 2px em `line` na base, para não parecer dado
  faltando.
- Eixos em 13px e `muted`, com datas como 03/11 e valores abreviados (R$ 1,2
  mil). Grade só horizontal, em `line-soft`.
- Tooltip em caixa flutuante, com a data e o valor do dia; a coluna sob o mouse
  ganha fundo `accent-soft`. As setas do teclado percorrem os dias.
- O melhor dia não ganha cor própria, porque já aparece em número ao lado do
  gráfico.

Marca:

- O nome "OrderFlow" em Plex Sans Condensed 700, 20px, branco na barra
  azul-marinho. Sem símbolo.
- Favicon: quadrado azul-marinho (`bar`) com um "O" branco na mesma fonte, em
  SVG, com a letra convertida em contorno para não depender da fonte. Tem
  contraste em aba clara e escura; a versão com o quadro azul-claro quase
  sumia em aba clara.

Status: quadrado colorido mais rótulo em português, nunca só a cor. Quadrado
vazado indica que algo ainda está pendente. Enquanto o polling do pagamento
roda, o marcador de "Processando" gira. No teto do polling, ele para, e o
bloco mostra "Sem resposta ainda" com o botão "Verificar de novo", porque
`PENDING` pode não terminar (§6.2).

| Status | Rótulo | Cor | Cor no escuro | Marcador |
|---|---|---|---|---|
| Pedido `PENDING` | Pendente | `#7A88A3` | `#8D9AB5` | vazado |
| Pedido `CONFIRMED` | Confirmado | `#2350D8` | `#8AA6FF` | cheio |
| Pedido `SHIPPED` | Enviado | `#0891B2` | `#3CC4E0` | cheio |
| Pedido `DELIVERED` | Entregue | `#2F855A` | `#4FBF82` | cheio |
| Pedido `CANCELLED` | Cancelado | `#A0AEC4` | `#6E7C97` | cheio |
| Pagamento `PENDING` | Processando | `#B7791F` | `#E0A845` | vazado; gira durante o polling |
| Pagamento `APPROVED` | Aprovado | `#2F855A` | `#4FBF82` | cheio |
| Pagamento `DECLINED` | Recusado | `#C53030` | `#F07272` | cheio |
| Pagamento `FAILED` | Falhou | `#8B2525` | `#C97070` | cheio |
| Notificação `PENDING` | Na fila | `#7A88A3` | `#8D9AB5` | vazado |
| Notificação `SENT` | Enviada | `#2F855A` | `#4FBF82` | cheio |
| Notificação `FAILED` | Falhou | `#8B2525` | `#C97070` | cheio |
| Notificação `NO_RECIPIENT` | Sem destinatário | `#B7791F` | `#E0A845` | cheio |
| Notificação `SUPERSEDED` | Substituída | `#A0AEC4` | `#6E7C97` | cheio |

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

### Modo escuro: cianotipia

À noite, a prancheta vira planta de engenharia: papel azul-marinho com traço
claro. A identidade continua a mesma; só os valores dos tokens mudam, sob
`[data-theme="dark"]`. O fundo das caixas usa quase o azul do texto do modo
claro.

- **Como alternar:** segue o tema do sistema (`prefers-color-scheme`). Um botão
  de tema no topo da loja e na barra lateral do backoffice abre Sistema, Claro
  e Escuro (no celular, o tema fica dentro do Menu), e a escolha fica no
  localStorage. Um script curto no `<head>`
  aplica o tema antes da primeira pintura, para a página não abrir no tema
  errado e trocar em seguida. Assim o sistema é respeitado e quem vê a demo
  troca de tema sem mexer no sistema.
- **Cobalto:** clareia para `#8AA6FF`, e o botão cheio passa a ter texto
  escuro. O foco usa `accent` em todo lugar.
- **Barra:** fica mais escura que o fundo e ganha borda `line` embaixo, porque
  só a cor separa pouco (1,07:1).
- **O que flutua:** diálogo, menu e calendário usam `raised`, mais claro que
  `surface`, porque sombra quase não aparece no escuro.
- **Contraste:** conferido sobre `surface` e `bg`. Todo texto passa de 4,5:1;
  todo marcador de status, de 3:1.

| Token | Escuro |
|---|---|
| `bg` | `#0D1830` |
| `surface` | `#13213F` |
| `raised` | `#1A2B50` |
| `ink` | `#E4EAF5` |
| `muted` | `#A3B1CC` |
| `line` | `#2C3F68` |
| `line-soft` | `#1F3054` |
| `bar` | `#081123` |
| `accent` | `#8AA6FF` |
| `on-accent` | `#0D1830` |
| `accent-soft` | `#1A2D5C`, grade do quadro em `#8AA6FF33` |
| `info-*` | fundo `#1A2D5C`, borda `#34508F`, texto `#C9D6FF`, marcador `#8AA6FF` |
| `success-*` | fundo `#12301F`, borda `#2E6B4C`, texto `#A9E5C4`, marcador `#4FBF82` |
| `warning-*` | fundo `#2E2410`, borda `#6B5220`, texto `#F0D08A`, marcador `#E0A845` |
| `danger-*` | fundo `#361A20`, borda `#7A3640`, texto `#F5B8B8`, marcador `#F07272` |
| `disabled` | texto `#7584A3` sobre `bg`, borda `line` |
| `shadow-float` | `0 12px 32px #00000080` |
| `overlay` | `#050B18B3` |

As cores de status no escuro estão na tabela de status, acima.

Descartado: **Grafite**, cinza-escuro neutro com azul só no destaque. Os
status se destacariam mais, mas é o modo escuro de qualquer app (preto e um
acento só), e a ficha técnica perderia o papel azul.

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

### Estados de tela: carregando, erro e falhas

Cada serviço do backend cai sozinho: com o catálogo fora, os pedidos continuam
(§7.9). Por isso a falha ocupa o menor pedaço possível da tela.

- **Carregando:** nada nos primeiros 300 ms; depois, um esqueleto com a forma
  da tela (cards, linhas, blocos), montado com peças comuns. O que não depende
  da API (topo, título, ordenação) já aparece pronto. Resposta rápida não
  pisca, e a tela não pula quando os dados chegam. O esqueleto pulsa devagar e
  fica parado com `prefers-reduced-motion`. O custo é de manutenção: quando o
  layout de uma tela muda, o esqueleto dela muda junto. Um indicador único no
  centro seria um componente só, mas a tela pularia ao carregar.
- **Trocando de página ou atualizando:** os dados antigos ficam na tela,
  esmaecidos, até os novos chegarem.
- **Repetição automática:** leitura que falha com 503, 504 ou erro de rede
  repete uma vez, depois de 1 s, e só então mostra o erro. Ação nunca repete
  sozinha, porque depois de um 504 ela pode ter acontecido (§3.9). Três
  repetições, o padrão do TanStack Query, aguentariam o circuit breaker aberto
  (10 s), mas deixariam a pessoa até 7 s olhando o esqueleto.
- **Erro num bloco:** o bloco diz o que não carregou e tem "Tentar de novo",
  de contorno; o resto da tela continua. É o que o bloco de pagamento já fazia.
- **Erro no dado principal:** sem o pedido ou o catálogo não há o que mostrar,
  então o erro ocupa a área da tela, com o topo e o menu no lugar. Aí "Tentar
  de novo" é a ação principal e vira o botão cheio.
- **Ação que falhou:** durante o envio, o botão fica desativado, com o marcador
  girando e o mesmo rótulo. Depois de 503, 504 ou erro de rede, o front relê o
  dado e diz o que encontrou ("o pedido continua pendente" ou "o pedido já foi
  confirmado"), porque a API grava antes de responder e a ação pode ter
  acontecido. Só repetir a mensagem de erro faria a pessoa refazer o que já
  estava feito. Erros 4xx (validação, conflito) seguem os textos de cada tela.
- **Sem conexão:** erro de rede abre uma faixa só, abaixo do topo, em
  `warning`: "Sem conexão com o servidor. A página tenta de novo sozinha."
  Enquanto ela está aberta, uma leitura leve a cada 10 s confere se o servidor
  voltou; quando volta, a faixa some e o que falhou é lido de novo. Os blocos
  dizem só o que não carregou, sem repetir a explicação. Risco: um erro gerado
  pelo próprio gateway pode chegar sem CORS e parecer erro de rede (§11, item
  20).
- **Sem acesso:** o link do backoffice só aparece para admin. Quem abre o
  endereço sem o papel vê "Esta área é só para administradores" com "Voltar à
  loja", em vez de voltar à loja sem saber por quê.
- **Página não encontrada:** uma tela só, "Página não encontrada", com "Ir para
  o catálogo", para endereço errado e para pedido ou produto que não existe.
  Um texto por tipo diria o que faltou; a tela genérica ficou por ser um caso
  só.
- **Sessão encerrada:** quando o refresh falha de vez, a pessoa vai para Entrar
  com "Sua sessão terminou. Entre de novo para continuar de onde parou" e,
  depois do login, volta à mesma tela, como no fluxo do carrinho. O que estava
  digitado num formulário se perde; um login em diálogo evitaria isso, mas
  teria de funcionar por cima de qualquer tela. 503 ou 504 no refresh não
  encerram a sessão: é o serviço de contas fora (§4.5).

### Celular

Três larguras: até 639px é celular, de 640 a 1023px é tablet, de 1024px em
diante é computador. Componente que muda pelo espaço que tem, como a linha do
carrinho, usa container query.

Loja:

- **Catálogo:** uma coluna no celular, com o card deitado: quadro de 96px à
  esquerda; nome, descrição e preço à direita; "Adicionar ao carrinho" largo
  embaixo. Cabem dois ou três produtos por tela, e o rótulo inteiro cabe. Duas
  colunas espremeriam nome e descrição em 160px; o card em pé mostraria um
  produto e meio. No tablet são duas colunas; no computador, três.
- **Navegação:** o topo mostra o carrinho com a contagem e o botão Menu, que
  abre por cima da tela Catálogo, Meus pedidos, Conta e o tema (Sistema, Claro,
  Escuro). Esc, o fundo ou o próprio Menu fecham. O carrinho fica sempre à
  vista, e nada fixo come a tela. Abas fixas embaixo deixariam tudo a um
  toque, mas disputariam espaço com a barra do navegador e com a de "Finalizar
  compra".
- **Barra fixa:** no carrinho e na revisão do pedido, o total e o botão
  principal ("Finalizar compra", "Confirmar compra") ficam presos ao rodapé,
  acima da área segura do aparelho (`env(safe-area-inset-bottom)`). Assim o
  total fica sempre à vista e o botão, ao alcance do polegar. "Cancelar
  pedido" fica no fim da lista, fora da barra.
- **Detalhe do produto:** quadro em cima, com a largura da tela; quantidade e
  "Adicionar ao carrinho" logo abaixo do preço.
- **Pedido:** o bloco de pagamento vem antes dos itens, porque é o que muda
  depois da compra. As etapas ficam na vertical: lado a lado, os rótulos se
  atropelam em 360px.
- **Tabelas da loja** (Meus pedidos, itens do pedido e da revisão) viram
  lista. Em Meus pedidos: código e status em cima, itens no meio, data e total
  embaixo.

Backoffice:

- Abaixo de 1024px, a barra lateral vira o Menu do topo.
- No celular, cada linha de tabela vira um card (Pedidos, Produtos,
  Notificações, Mais vendidos), e os botões das linhas voltam a ter 44px. É um
  segundo layout por tabela, mas deixa o backoffice usável numa demo no
  celular. Tabela rolando de lado foi descartada; no tablet, a tabela continua.
- Botão desativado ganha uma linha que explica por quê ("Liberado quando o
  pagamento for aprovado").
- Vendas: datas lado a lado, números em grade de dois, e o gráfico rola de
  lado, aberto no dia mais recente.

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

- CI e deploy.
