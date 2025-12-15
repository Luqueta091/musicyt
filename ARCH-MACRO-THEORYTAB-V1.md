Arquitetura Macro v1.0

Documento ID: ARCH-MACRO-THEORYTAB-V1
Sistema: Theory Tab Player
Data: 2025-12-15
Status: Draft

Visão do sistema

Objetivo da V1: entregar um player web simples onde o aluno escolhe música e instrumento e entende visualmente teoria (graus 1–7 + romanos) sincronizada ao vídeo.

Foco operacional da V1: quase tudo manual (sem jobs complexos).

Metodologia organizacional
Modular Domain Layered Architecture

Módulos por domínio (Identity, Music Catalog, Theory Tabs)

Camadas por módulo (controllers → services → repository/entities/dtos) mantendo separação mesmo no MVP

Shared só para globais (infra/util, sem regra de negócio)

Casos de uso não conhecem HTTP/SQL/framework (controller traduz, use case orquestra).

C4 — Contexto
flowchart LR
  U[Aluno] --> W[Web App<br/>Theory Tab Player]
  A[Admin] --> W

  W --> YT[YouTube Embed]
  W --> API[Backend API]

  API --> DB[(Database)]

  style W fill:#e3f2fd
  style API fill:#e8f5e9
  style DB fill:#fff3e0
  style YT fill:#fce4ec


Notas

YouTube entra na V1 “ao menos embed”.

O diferencial é a sincronização e visualização no player.

C4 — Containers
flowchart TB
  subgraph Client["Client"]
    WEB["Web App<br/>(Biblioteca + Player + Admin simples)"]
  end

  subgraph Server["Server"]
    API["HTTP API<br/>(modular + camadas)"]
  end

  subgraph Data["Data"]
    DB[(Relational DB)]
  end

  subgraph External["External"]
    YT["YouTube (Embed)"]
  end

  WEB -->|HTTP| API
  API -->|SQL| DB
  WEB -->|iFrame/Embed| YT

C4 — Componentes do Backend API
flowchart TB
  subgraph API["Backend API"]
    direction TB

    subgraph Shared["shared/"]
      ERR[errors]
      MID[middlewares]
      UTIL[utils/result]
      DBCL[db client]
    end

    subgraph Identity["modules/identity"]
      IC[controllers]
      IS[services]
      IR[repository]
      IE[entities]
      ID[dtos]
    end

    subgraph Music["modules/music-catalog"]
      MC[controllers]
      MS[services]
      MR[repository]
      ME[entities]
      MD[dtos]
    end

    subgraph Tabs["modules/theory-tabs"]
      TC[controllers]
      TS[services]
      TR[repository]
      TE[entities]
      TD[dtos]
    end

    IC --> IS --> IR --> DBCL
    MC --> MS --> MR --> DBCL
    TC --> TS --> TR --> DBCL

    IS --> UTIL
    MS --> UTIL
    TS --> UTIL
    IC --> ERR
    MC --> ERR
    TC --> ERR
    IC --> MID
    MC --> MID
    TC --> MID
  end

Mapa de módulos
MOD-001 Identity

Responsabilidade única: autenticar (ou convidado) e identificar aluno/admin.

O que entra

Login (email/senha)

“Continue as guest” (sessão efêmera no client; opcional backend)

User.tipo = aluno/admin

O que não entra (V1)

Login social (V2)

Perfis criador/moderador (V2)

MOD-002 Music Catalog

Responsabilidade única: biblioteca de músicas + busca.

O que entra

CRUD de música (pelo admin)

Busca por título/artista

Expor se existem trilhas por instrumento (derivado de Theory Tabs)

MOD-003 Theory Tabs

Responsabilidade única: gerenciar TheoryTab e servir dados pro player.

O que entra

CRUD de TheoryTab, TrilhaDeInstrumento, BlocoTeorico

Status lifecycle (rascunho, em_revisao, publicado, arquivado)

Endpoint otimizado pro player: “me dá a trilha do instrumento X com blocos ordenados”

Regras de comunicação e dependências

Theory Tabs referencia Music por ID (FK no banco).

Web App nunca fala com DB, só com API.

Use case não conhece HTTP/SQL/framework — controller adapta e repository isola persistência.

Shared não pode virar lixeira: service em shared só se for infraestrutura pura (ex.: email/logger/cache).

Convenções e estrutura de pastas

Vou seguir a referência mínima do guia:

Backend:

src/shared/

src/modules/[modulo]/controllers|services|repository|entities|dtos|index.ts|README.md

Frontend:

src/shared/

src/modules/[rota]/...

Os 5 princípios (sem exceção): profundidade, separação, convenção, espelhamento, shared só globais.

Fluxos principais
Fluxo 1 — Aluno estuda uma música

Baseado no teu fluxo do aluno (biblioteca → música → instrumento → play/pause → troca instrumento).

sequenceDiagram
  participant U as Aluno
  participant W as Web App
  participant API as Backend API
  participant YT as YouTube Embed

  U->>W: Abre biblioteca
  W->>API: GET /musicas?search=...
  API-->>W: Lista músicas + instrumentos disponíveis

  U->>W: Escolhe música + instrumento
  W->>API: GET /musicas/{id}/player?instrumento=piano
  API-->>W: Dados (youtube_id + trilha + blocos ordenados)

  W->>YT: Carrega embed e dá play
  loop a cada tick
    W->>YT: read currentTime
    W->>W: destaca bloco(s) ativos por tempo
  end

  U->>W: Troca instrumento
  W->>API: GET /musicas/{id}/player?instrumento=guitarra
  API-->>W: Nova trilha + blocos

Fluxo 2 — Admin cadastra conteúdo V1

Objetivo: você conseguir cadastrar sem mexer em código.

sequenceDiagram
  participant A as Admin
  participant W as Admin UI (simples) / Script
  participant API as Backend API
  participant DB as Database

  A->>W: Cria Música (título/artista/youtube)
  W->>API: POST /admin/musicas
  API->>DB: INSERT musica

  A->>W: Cria TheoryTab (status=racsunho)
  W->>API: POST /admin/theory-tabs
  API->>DB: INSERT theory_tab

  A->>W: Cria Trilha por instrumento
  W->>API: POST /admin/theory-tabs/{id}/trilhas
  API->>DB: INSERT trilha_instrumento

  A->>W: Sobe blocos (bulk)
  W->>API: PUT /admin/trilhas/{id}/blocos (array)
  API->>DB: UPSERT blocos

Estrutura do repositório

Abaixo vai uma proposta monorepo (porque você tem Web + API desde a V1). Não é “tecnologia”, é organização.

repo-root/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── shared/
│   │   │   │   ├── db/
│   │   │   │   ├── errors/
│   │   │   │   ├── middlewares/
│   │   │   │   └── utils/
│   │   │   ├── modules/
│   │   │   │   ├── identity/
│   │   │   │   │   ├── controllers/
│   │   │   │   │   ├── services/
│   │   │   │   │   ├── repository/
│   │   │   │   │   ├── entities/
│   │   │   │   │   ├── dtos/
│   │   │   │   │   ├── interfaces/
│   │   │   │   │   ├── events/
│   │   │   │   │   ├── index.ts
│   │   │   │   │   └── README.md
│   │   │   │   ├── music-catalog/
│   │   │   │   │   ├── controllers/
│   │   │   │   │   ├── services/
│   │   │   │   │   ├── repository/
│   │   │   │   │   ├── entities/
│   │   │   │   │   ├── dtos/
│   │   │   │   │   ├── interfaces/
│   │   │   │   │   ├── events/
│   │   │   │   │   ├── index.ts
│   │   │   │   │   └── README.md
│   │   │   │   └── theory-tabs/
│   │   │   │       ├── controllers/
│   │   │   │       ├── services/
│   │   │   │       ├── repository/
│   │   │   │       ├── entities/
│   │   │   │       ├── dtos/
│   │   │   │       ├── interfaces/
│   │   │   │       ├── events/
│   │   │   │       ├── index.ts
│   │   │   │       └── README.md
│   │   │   ├── routes.ts
│   │   │   └── main.ts
│   │   └── README.md
│   └── web/
│       ├── src/
│       │   ├── shared/
│       │   │   ├── components/
│       │   │   ├── hooks/
│       │   │   ├── contexts/
│       │   │   └── utils/
│       │   ├── modules/
│       │   │   ├── library/
│       │   │   │   ├── components/
│       │   │   │   ├── hooks/
│       │   │   │   ├── index.tsx
│       │   │   │   └── README.md
│       │   │   ├── player/
│       │   │   │   ├── components/
│       │   │   │   ├── hooks/
│       │   │   │   ├── index.tsx
│       │   │   │   └── README.md
│       │   │   ├── admin/
│       │   │   └── auth/
│       │   ├── App.tsx
│       │   └── routes.tsx
│       └── README.md
└── docs/
    └── architecture/
        ├── ARCH-MACRO-THEORYTAB-V1.md
        ├── ARCH-IDENTITY-V1.md
        ├── ARCH-MUSIC-CATALOG-V1.md
        ├── ARCH-THEORY-TABS-V1.md
        └── ARCH-DB-V1.md


Esse shape bate com a “Estrutura Backend/Frontend mínima” do guia (separando shared/ e modules/ e mantendo previsibilidade).

Script rápido pra gerar as pastas

Se você quiser materializar isso rápido:

# backend modules
mkdir -p apps/api/src/shared/{db,errors,middlewares,utils}
for m in identity music-catalog theory-tabs; do
  mkdir -p apps/api/src/modules/$m/{controllers,services,repository,entities,dtos,interfaces,events}
  touch apps/api/src/modules/$m/{index.ts,README.md}
done
touch apps/api/src/{main.ts,routes.ts}

# frontend modules
mkdir -p apps/web/src/shared/{components,hooks,contexts,utils}
for r in library player admin auth; do
  mkdir -p apps/web/src/modules/$r/{components,hooks}
  touch apps/web/src/modules/$r/{index.tsx,README.md}
done
touch apps/web/src/{App.tsx,routes.tsx}
