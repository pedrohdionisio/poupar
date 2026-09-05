> 🌎 [English](README.en.md) · **Português (Brasil)**

# Poupar

Controle de gastos de supermercado que transforma a **foto de um cupom fiscal** em compra
estruturada, com histórico de preço por produto e gasto por categoria.

Este é o repositório **hub** do projeto: ele não tem código, só reúne os repositórios que formam o
Poupar e explica como eles se encaixam.

---

## Repositórios

| Repositório | O que é | Stack |
| --- | --- | --- |
| [**poupar-app**](https://github.com/pedrohdionisio/poupar-app) | App mobile — login, recibos, estatísticas, scan de cupom, compra manual e estabelecimentos. | React Native 0.86 + Expo 57, TypeScript estrito, NativeWind, TanStack Query |
| [**poupar-api**](https://github.com/pedrohdionisio/poupar-api) | API serverless — auth, compras, scans, produtos e agregações. | AWS Lambda + API Gateway HTTP v2, DynamoDB single-table, Cognito, S3, SQS, OpenAI |

Cada repositório tem o próprio README completo (arquitetura, convenções, variáveis de ambiente,
deploy) em português e em inglês.

---

## Como as peças se encaixam

```
┌──────────────────┐    HTTPS + JWT (Cognito)    ┌──────────────────────────┐
│   poupar-app     │ ──────────────────────────▶ │       poupar-api         │
│  React Native    │ ◀────────────────────────── │  Lambda + API Gateway    │
└────────┬─────────┘                             └────────────┬─────────────┘
         │                                                    │
         │ upload direto (presigned POST)                     │
         ▼                                                    ▼
   ┌───────────┐   ObjectCreated   ┌───────────┐      ┌────────────────┐
   │  S3       │ ────────────────▶ │  SQS      │ ───▶ │  DynamoDB      │
   │  uploads  │                   │  scans    │      │  single-table  │
   └───────────┘                   └─────┬─────┘      └────────────────┘
                                         │
                                         ▼
                                   ┌───────────┐
                                   │  OpenAI   │  extração do cupom
                                   └───────────┘
```

O app **nunca** fala com S3, SQS ou DynamoDB por conta própria: a única exceção é o upload da foto,
que vai direto para o S3 com uma assinatura emitida pela API.

---

## Fluxo de scan de cupom, ponta a ponta

O caminho que atravessa os dois repositórios:

| # | Onde | O que acontece |
| --- | --- | --- |
| 1 | app | Usuário escolhe o estabelecimento e fotografa o cupom (câmera ou galeria; HEIC vira JPEG no iOS). |
| 2 | app → api | `POST /scans` devolve `scanId` e um presigned POST válido por 5 min. |
| 3 | app → S3 | Upload multipart direto, em uma instância axios sem `Authorization` — o S3 recusa o header. |
| 4 | api | `ObjectCreated` no prefixo `scans/` publica na SQS; a lambda `processScan` extrai os itens via OpenAI e grava o rascunho. |
| 5 | app → api | Polling em `GET /scans/{scanId}` a cada 2s, teto de 180s: `PENDING → PROCESSING → AWAITING_REVIEW` (ou `FAILED`). |
| 6 | app → api | Usuário revisa o rascunho e confirma: `POST /scans/{scanId}/confirm` cria a compra, o recibo, os produtos, os price points e atualiza o gasto por categoria. |

---

## Rodando o ecossistema

A ordem importa: **o app não tem mock de backend**, então a API precisa estar deployada antes.

### 1. API

Requer Node 22+, pnpm 11+, credenciais AWS, Serverless Framework v4 autenticado, uma chave da OpenAI
e um domínio verificado no SES.

```bash
git clone git@github.com:pedrohdionisio/poupar-api.git
cd poupar-api
pnpm install
cp .env.example .env      # preencha antes de seguir
serverless deploy --stage <seu-stage>
```

Anote a URL do API Gateway que o deploy imprime.

### 2. App

Requer Node 20+, yarn 1.22 e Xcode 16+ e/ou Android Studio. O projeto usa `expo-dev-client` — o Expo
Go **não** roda este app.

```bash
git clone git@github.com:pedrohdionisio/poupar-app.git
cd poupar-app
yarn install
cp .env.example .env      # EXPO_PUBLIC_API_URL = a URL do passo anterior
yarn ios                  # ou: yarn android
```

Com o dev client já instalado, o dia a dia é só `yarn start`.

---

## Convenções compartilhadas

Os dois repositórios são independentes, mas seguem o mesmo espírito:

- **TypeScript estrito**, sem `any` e sem `as` para calar o compilador.
- **Dependência aponta numa direção só** — Clean Architecture na API
  (`main → application → entities`), três camadas no app (`presentation → data → shared`).
- **Dinheiro em centavos inteiros e quantidade em milésimos** no contrato da API; a conversão para
  reais e unidades mora só nos mappers do app.
- **Comentário explica o porquê**, nunca repete o código.
- **Named export** como padrão; Biome cuidando de lint, formatação e ordem de imports.
- Cada repositório carrega o próprio ferramental de IA em `.claude/` (rules, skills e subagents).

---

## Licença

`poupar-api` é ISC. `poupar-app` é privado, sem licença de distribuição definida.
