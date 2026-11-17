# Diozz Atacado — Especificação Arquitetural, Infraestrutura e Escopo do MVP

> **Cliente:** Diozz — Loja de Roupas para Atacado (Fortaleza/CE)  
> **Catálogo Atual (Legado):** `https://facilzap.app.br/diozz`  
> **Objetivo:** Plataforma proprietária **Mobile-First**, minimalista, de alta conversão para atacado, com sincronização inicial via API da FácilZap (Fase 1) e evolução para sistema próprio completo e escalável (Fase 2).

---

## 1. Visão Geral e Estratégia de Entrega em Fases

Para não interromper a operação atual da loja nem exigir retrabalho imediato da equipe, o projeto adota uma **estratégia evolutiva em duas fases**:

| Fase | Foco | Fonte de Dados | Benefício Imediato para a Diozz |
| :--- | :--- | :--- | :--- |
| **Fase 1 — Vitrine Própria + Motor Híbrido** | Novo Front-end Mobile-First + Carrinho Persistente + Filas BullMQ (Carrinho Abandonado) + Notificações Globais | **API REST FácilZap (`api.facilzap.app.br`) + Webhooks** + Cache Local (Redis/PostgreSQL) | Zero mudança na rotina da equipe; aumento imediato de conversão, velocidade, SEO e recuperação automática de carrinhos. |
| **Fase 2 — Independência Total & Painel Escalável** | Painel Admin Próprio + Controle de Estoque + Pedidos por Etapas + Impressão de Etiqueta/Romaneio | **Banco PostgreSQL Próprio** (migração assistida da FácilZap) | Fim da dependência e limites da FácilZap; regras avançadas de atacado, romaneio com 1 clique e operação 100% sob controle. |

---

## 2. Stack Tecnológica Definida

A arquitetura foi desenhada para entregar velocidade instantânea no 4G/5G (público atacadista navegando pelo celular) e alta escalabilidade no back-end:

### Front-end (Storefront & Painel Administrativo)
* **Framework:** **Next.js (App Router)** com **React** e **TypeScript**
  * *Por que:* Renderização híbrida (SSR/ISR) garantindo carregamento em milissegundos, URLs amigáveis (`/produto/regata-marrom`) e indexação no Google (SEO).
* **Estilização & UI:** **Tailwind CSS** + **Framer Motion** (ou transições CSS nativas ultra-leves)
  * *Por que:* Visual minimalista, *mobile-first*, com micro-animações limpas e fluidas ao adicionar/remover variações no carrinho.
* **Estado Global & Persistência:** **Zustand** com middleware `persist` (`localStorage` + sincronização assíncrona no back-end via token anônimo de sessão)
  * *Por que:* O carrinho permanece intacto mesmo que o cliente feche o navegador, mude de aplicativo ou recarregue a página.
* **PWA (Progressive Web App):** Manifesto + *Service Worker* para que revendedores instalem o catálogo na tela inicial do celular.

### Back-end & Processamento Assíncrono
* **Runtime & API:** **Node.js** com **Fastify** ou **NestJS** (TypeScript ponta a ponta)
  * *Por que:* Alta vazão de requisições simultâneas, tipagem compartilhada com o front-end e facilidade de integração com APIs externas.
* **Cache & Filas:** **Redis** + **BullMQ**
  * *Cache:* Armazena catálogo sincronizado da FácilZap, contadores de visitas em tempo real e sessão do usuário.
  * *Filas (BullMQ):* Gerencia disparos de recuperação de carrinho abandonado (após `X` minutos configuráveis), processamento de webhooks e notificações.
* **Tempo Real (Notificação Global):** **Server-Sent Events (SSE)** ou **WebSockets (Socket.io)**
  * *Por que:* Envia em tempo real o alerta global (*"Cliente X de Fortaleza-CE acabou de comprar 12 peças"*) para todos os usuários conectados no catálogo.

### Banco de Dados & Infraestrutura
* **Banco Principal:** **PostgreSQL** (com **Prisma ORM**) — modelagem relacional para produtos, variações/grades, estoque, pedidos, perguntas, avaliações e logs.
* **Armazenamento de Mídia (Fotos/Vídeos):** **Cloudflare R2** ou **AWS S3** + **CDN** com conversão automática para `WebP/AVIF`.
* **Hospedagem Recomendada:**
  * Front-end Next.js: **Vercel** ou **Container Docker em VPS dedicada (Hetzner / AWS / DigitalOcean com Coolify)**.
  * Back-end Node.js + PostgreSQL + Redis: **VPS Dedicada / Cloud gerenciada** com backups automáticos diários, SSL (HTTPS) e monitoramento.

---

## 3. Arquitetura do Sistema (Diagrama Lógico)

```text
+-------------------------------------------------------------------+
|                   CLIENTES (MOBILE-FIRST / PWA)                   |
|  Catálogo -> Slug Produto (+ / - Direto) -> Carrinho Persistente  |
+-------------------+---------------------------+-------------------+
                    | HTTPS (REST / SSR)        | SSE / WebSocket
                    v                           v
+-------------------------------------------------------------------+
|                 API GATEWAY / BACK-END (Node.js)                  |
|  - Catálogo & Slug (Visitas/Vendas)   - Checkout Enxuto           |
|  - Engine de Notificação Global       - Painel Admin & Romaneio   |
+---------+--------------------+--------------------+---------------+
          |                    |                    |
          v                    v                    v
+-------------------+  +---------------+  +-------------------------+
|    PostgreSQL     |  | Redis + BullMQ|  |   Integrações Externas  |
| - Produtos/Grades |  | - Cache API   |  | - API FácilZap (Fase 1) |
| - Pedidos/Etapas  |  | - Fila:       |  | - WhatsApp API (Alertas |
| - Perguntas/Aval. |  |   Carrinho    |  |   e Recup. de Carrinho) |
| - Configs Painel  |  |   Abandonado  |  | - Pagamento / Frete     |
+-------------------+  +---------------+  +-------------------------+
```

---

## 4. Requisitos Funcionais Detalhados do MVP

### 4.1. Experiência da Loja (Vitrine Editorial Minimalista & Mobile-First)
* **Estética Farfetch (Clean & Sofisticada):** Fundo branco, tipografia geométrica editorial (`Plus Jakarta Sans` / grotesca limpa), grid sem bordas pesadas e **ícones lineares SVG minimalistas** (zero emojis). Sem previsão de entrega no slug para manter foco total na peça.
* **Home 100% Clean:** Foco direto nos cards de produtos sem banners pesados ocupando a primeira dobra.
* **Cabeçalho Limpo:** Logotipo oficial Diozz (ao clicar, redireciona para a Home e limpa filtros), barra de busca instantânea (*live search*), **Minha Conta** e ícone do Carrinho com contador dinâmico animado.
* **Navegação por Categorias no Mobile:** Barra horizontal com **setas direcionais (`‹` / `›`)** e **menu em cascata ("Categorias ▾")** para navegação ágil em telas pequenas.
* **Zoom nos Cards:** Efeito suave de *zoom* na foto/mídia ao passar o mouse sobre os cards de produto.
* **3 Sugestões de Paleta de Cores:** Verde Floresta (confiança atacado), Preto Editorial (luxo minimalista) e Azul Marinho (estilo litoral/Fortaleza).
* **Botão Flutuante WhatsApp:** Botão fixo no canto inferior direito para contato imediato com a equipe Diozz.
* **Notificação Global em Tempo Real:**
  * Sempre que um pedido é finalizado (ou recebido via webhook), um *toast* discreto e elegante aparece na tela dos visitantes ativos (*Social Proof*), configurável no painel (tempo de exibição, texto, ativar/desativar).

### 4.2. Página do Produto (`/[slug]`) — Inspirada na Referência Diozz
Baseada diretamente no layout aprovado pelo cliente:
1. **Galeria de Mídia (Fotos + Vídeos MP4):** Miniaturas laterais (desktop) / inferiores (mobile), suporte nativo a **vídeos de produto (`.mp4`)** com reprodução automática silenciosa/controles, setas de navegação, zoom no hover e botão de download da mídia (essencial para revendedores divulgarem).
2. **Prova Social & Métricas:** Indicadores no topo (`👁 162 visitas` · `🛒 9 vendas`), atualizados via Redis sem pesar o banco.
3. **Cabeçalho do Produto:** Título, tags de categoria/destaque, preço unitário em destaque e simulação de parcelamento / Pix.
4. **Seleção de Variações Direto para o Carrinho (`+` e `-`):**
   * Lista clara de tamanhos (`P`, `M`, `G`, `GG` e variações de cor).
   * **Regra solicitada:** Ao clicar em **`+`**, o item entra **imediatamente** no carrinho (sem precisar clicar em um segundo botão obrigatório), exibindo o número da quantidade no seletor e disparando uma **animação *clean*** (pulso sutil no card da variação + salto suave no ícone do carrinho + feedback visual).
   * Clicar em **`-`** reduz ou remove imediatamente do carrinho com transição suave.
   * **Persistência:** Estado salvo em `localStorage` + `cart_session_id`. Ao fechar e reabrir o navegador, tudo continua lá.
5. **Seções Complementares na Página do Produto:**
   * **Descrição Técnica:** Composição do tecido, gramatura, modelagem e guia de medidas.
   * **Produtos Relacionados:** Carrossel *mobile-first* de peças complementares (ex.: combinando regata com short).
   * **Perguntas e Respostas:** Área para dúvidas públicas sobre tecido/grade, respondidas pelo painel.
   * **Avaliações com Comentários:** Nota média em estrelas e depoimentos de compradores verificados.

### 4.3. Área do Cliente & Checkout Enxuto
* **Sem barreiras de cadastro:** Navegação e montagem de carrinho totalmente abertas.
* **Checkout em 1 Tela (Apenas Nome e Telefone):**
  * Solicita exclusivamente **Nome Completo** e **WhatsApp (Telefone)** no momento de fechar o pedido.
  * Seleção de entrega (Retirada / Excursão em Fortaleza / Correios ou Transportadora) e forma de pagamento.
* **Área "Minha Conta" Enxuta:**
  * Acesso rápido informando o **Telefone** (com validação simples ou link único via WhatsApp).
  * Visualização do histórico de pedidos e **Linha do Tempo do Pedido** (ex.: `Recebido` -> `Em Separação` -> `Faturado/Embalado` -> `Enviado`).

---

## 5. Automação de Carrinho Abandonado (Redis + BullMQ)

Fluxo técnico para recuperar vendas sem intervenção manual:

1. Usuário adiciona itens ao carrinho e informa seu telefone (no pré-checkout ou sessão identificada).
2. O front-end envia um *heartbeat* ou evento `cart.updated` para o back-end Node.js.
3. O back-end agenda um *job* no **BullMQ** (`abandoned-cart-queue`) com atraso configurável no painel (ex.: `delay: 30 minutos`, `jobId: cart:{sessionId}`).
4. Se o usuário continuar editando o carrinho, o *job* anterior é substituído/reagendado (*debounce*).
5. Se o usuário **finalizar o pedido**, o *job* é **removido da fila** imediatamente.
6. Se o tempo `X` expirar e o pedido não tiver sido fechado:
   * O **Worker BullMQ** dispara uma mensagem personalizada via **WhatsApp API** com o nome do cliente, resumo das peças reservadas e link direto que restaura o carrinho exato ao clicar.

---

## 6. Painel Administrativo Altamente Configurável e Escalável (2 Níveis de Acesso)

Projetado com **barra lateral de navegação e filtros**, com separação estrita de permissões entre **Administrador** e **Vendedor**:

* **Controle de Acesso (RBAC):**
  * **Perfil Administrador:** Acesso integral a todas as abas, incluindo faturamento, gráficos financeiros, cadastro de produtos, ajustes de estoque e configurações da loja.
  * **Perfil Vendedor:** Acesso restrito estritamente operacional (**Pedidos & Romaneio**, consulta de **Produtos** e consulta de **Estoque**). **Não visualiza faturamento, ticket médio ou totais financeiros.**

* **Aba Vendas & Faturamento (Dashboard Analítico):**
  * **Filtros na Barra Lateral:** Filtro por **Período** (Hoje, Últimos 7 dias, Últimos 30 dias, Personalizado) e por **Método de Pagamento** (Todos, Pix, Cartão de Crédito, Boleto/Link).
  * **Gráfico de Faturamento por Dia** e tabela detalhada com pedidos, receita diária e ticket médio.
* **Aba Produtos & Vídeos (Fase 1 Sync / Fase 2 Nativo):**
  * Cadastro simplificado de produto com múltiplas **fotos + vídeos `.mp4`**, tabela de variações (P/M/G/GG), SKU e preço de atacado.
* **Aba Estoque por Grade:**
  * Controle em tempo real por produto e tamanho (`P / M / G / GG`), ajuste rápido e alertas visuais de **Estoque Baixo**.
* **Aba Pedidos & Etapas + Romaneio (1 Clique):**
  * Mudança rápida de etapa com 1 clique (`Novo` -> `Pago` -> `Em Separação` -> `Despachado`), atualizando a **Área do Cliente** em tempo real.
  * Botão direto dentro do pedido para gerar **Etiqueta / Romaneio (10x15cm ou A4)** com Logo Diozz, código do pedido, cliente, itens por tamanho e envio/excursão.
* **Aba Configurações (White-Label):**
  * Troca de paleta de cores, logo, WhatsApp oficial, tempo `X` da fila BullMQ de carrinho abandonado e ativação da notificação global SSE.

---

## 7. Estratégia de Integração com a API FácilZap (Fase 1)

* **Leitura de Catálogo:** Um *Sync Worker* (cron no BullMQ a cada `N` minutos ou sob demanda) consulta `GET https://api.facilzap.app.br/produtos` usando `Authorization: Bearer {TOKEN}` e popula o cache no Redis/PostgreSQL para que a vitrine nunca sofra com lentidão externa.
* **Recebimento de Pedidos (Tempo Real):** Webhook da FácilZap (`pedido_criado`) conectado ao endpoint `POST /webhooks/facilzap` do nosso servidor Node.js para alimentar instantaneamente a **Notificação Global** e o histórico.
* **Envio de Novos Pedidos:** Pedidos finalizados na vitrine nova podem ser registrados na FácilZap via API (`POST /pedidos`) e/ou encaminhados formatados ao WhatsApp das vendedoras, mantendo o estoque unificado durante a transição.

---

## 8. Roadmap Sugerido de Execução

1. **Semana 1 — Protótipo Validado & Infraestrutura Base:** Setup Next.js + Node.js + PostgreSQL + Redis, definição final da identidade visual e integração de leitura com a API FácilZap.
2. **Semana 2 — Vitrine & Página de Produto (`/[slug]`):** Catálogo mobile-first, galeria, variações `+`/`-` direto no carrinho com animações e persistência local, avaliações e perguntas.
3. **Semana 3 — Checkout Enxuto, Filas & Tempo Real:** Área do cliente (Nome + Telefone), notificação global via SSE, fila BullMQ de carrinho abandonado e integração WhatsApp.
4. **Semana 4 — Painel Administrativo & Romaneio:** Gestão de pedidos por etapas, impressão de etiqueta/romaneio e configurações da loja. Go-live assistido da Fase 1!
