Arquitetura Micro por módulo

Abaixo, um documento completo pra cada módulo backend (os 3 módulos V1).

Arquitetura Micro Identity v1.0

Documento ID: ARCH-IDENTITY-V1
Módulo: Identity
Bounded Context: Identidade e Acesso
Data: 2025-12-15
Baseado em: ARCH-MACRO-THEORYTAB-V1
Status: Draft

Visão geral do módulo

Responsabilidade única (SRP): autenticar usuário (aluno/admin) e expor identidade da sessão.

Por que existe

V1 pode ter cadastro/login simples ou convidado

Admin pode existir “interno” (seed no banco)

Capacidades principais
Operação	Tipo	Descrição	Input	Output
Login	Command	Autentica por credenciais simples	LoginInputDTO	SessionOutputDTO
GetMe	Query	Retorna usuário da sessão	GetMeInputDTO	UserOutputDTO
Logout	Command	Invalida sessão/token	LogoutInputDTO	OkDTO

Observação: “Guest” pode ser 100% client-side na V1. Se quiser guest no backend, vira “CreateGuestSession”.

Arquitetura interna de camadas
graph TB
  subgraph "Identity"
    C[controllers]
    S[services]
    E[entities]
    R[repository]
    D[dtos]
    I[interfaces]
  end

  C --> D
  C --> S
  S --> E
  S --> I
  R -.-> I


Regra: caso de uso não conhece HTTP/SQL/framework.

Layer controllers

Responsabilidade

Validar formato (ex.: email string)

Traduzir HTTP → DTO

Chamar service

Mapear erro → resposta

Não faz

regra de negócio de autenticação “complexa”

query direta

Layer services

Casos de uso

LoginService.execute(input)

GetMeService.execute(input)

LogoutService.execute(input)

Guideline

Retornar Result (sucesso/erro) e não exceção de negócio.

Domain entities

User

id

nome

email

senha_hash (se login por senha)

tipo (aluno/admin)

Regras

email único

tipo controlado (enum)

Repository e interfaces

Interface

IUserRepository.findByEmail(email)

IUserRepository.findById(id)

IUserRepository.save(user)

Implementação

UserRepositoryDb faz mapping persistência ↔ domínio.

Estrutura de arquivos do módulo
src/modules/identity/
  controllers/
    auth.controller.ts
  services/
    login.service.ts
    get-me.service.ts
    logout.service.ts
  repository/
    user.repository.ts
    user.repository.db.ts
  entities/
    user.entity.ts
  dtos/
    login.dto.ts
    session-output.dto.ts
    user-output.dto.ts
  interfaces/
    user-repository.interface.ts
  events/
    user-logged-in.event.ts
  index.ts
  README.md

Arquitetura Micro Music Catalog v1.0

Documento ID: ARCH-MUSIC-CATALOG-V1
Módulo: Music Catalog
Bounded Context: Biblioteca de Músicas
Data: 2025-12-15
Baseado em: ARCH-MACRO-THEORYTAB-V1
Status: Draft

Visão geral do módulo

Responsabilidade única (SRP): gerenciar músicas e expor catálogo pesquisável pro player.

Capacidades principais
Operação	Tipo	Descrição	Input	Output
CreateMusic	Command	Admin cria música	CreateMusicDTO	MusicOutputDTO
UpdateMusic	Command	Admin edita metadados	UpdateMusicDTO	MusicOutputDTO
ListMusic	Query	Lista + busca por título/artista	ListMusicDTO	MusicListDTO
GetMusic	Query	Detalhe de música	GetMusicDTO	MusicOutputDTO
Arquitetura de camadas
graph TB
  C[controllers] --> S[services]
  S --> R[repository]
  S --> E[entities]
  C --> D[dtos]
  R --> DB[(db)]

Domain entities

Music (V1)

titulo, artista, youtube_url, tom_original, modo

Regras

titulo e artista obrigatórios

youtube_url válido (ou youtube_id normalizado)

busca por titulo/artista precisa índice

Repository

IMusicRepository.search(query)

IMusicRepository.findById(id)

IMusicRepository.save(music)

IMusicRepository.update(music)

Observação crítica de boundary

“Instrumentos disponíveis” aparece na biblioteca.
Isso depende de existirem trilhas no módulo Theory Tabs.

Pra evitar acoplamento tosco, duas abordagens limpas:

Query composta no backend: endpoint do catálogo faz join/aggregate com trilhas (read model)

Campo derivado/cached: music.available_instruments atualizado quando trilha muda (V1 pode ser 1)

Na V1 eu faria (1) porque é simples e não cria job.

Estrutura de arquivos do módulo
src/modules/music-catalog/
  controllers/
    music.controller.ts
  services/
    create-music.service.ts
    update-music.service.ts
    list-music.service.ts
    get-music.service.ts
  repository/
    music.repository.ts
    music.repository.db.ts
  entities/
    music.entity.ts
  dtos/
    create-music.dto.ts
    update-music.dto.ts
    list-music.dto.ts
    music-output.dto.ts
  interfaces/
    music-repository.interface.ts
  events/
    music-created.event.ts
    music-updated.event.ts
  index.ts
  README.md

Arquitetura Micro Theory Tabs v1.0

Documento ID: ARCH-THEORY-TABS-V1
Módulo: Theory Tabs
Bounded Context: Conteúdo Teórico Sincronizado
Data: 2025-12-15
Baseado em: ARCH-MACRO-THEORYTAB-V1
Status: Draft

Visão geral do módulo

Responsabilidade única (SRP): armazenar TheoryTab por música e servir trilhas/blocos sincronizáveis pelo player.

Core do domínio

TheoryTab é central e tem lifecycle (rascunho → revisão → publicado → arquivado).

Trilha por instrumento e blocos por tempo (início/duração).

Capacidades principais
Operação	Tipo	Descrição	Input	Output
CreateTheoryTab	Command	Cria tab p/ música	CreateTheoryTabDTO	TheoryTabDTO
SetTheoryTabStatus	Command	muda status	SetStatusDTO	TheoryTabDTO
UpsertInstrumentTrack	Command	cria/edita trilha (instrumento)	UpsertTrackDTO	TrackDTO
ReplaceBlocksBulk	Command	substitui blocos de uma trilha (bulk)	ReplaceBlocksDTO	TrackBlocksDTO
GetPlayerTrack	Query	dados pro player (instrumento)	GetPlayerTrackDTO	PlayerTrackDTO
Arquitetura de camadas
graph TB
  subgraph "Theory Tabs"
    C[controllers]
    S[services]
    R[repository]
    E[entities]
    D[dtos]
    I[interfaces]
  end

  C --> D
  C --> S
  S --> E
  S --> I
  R -.-> I
  R --> DB[(db)]

Regras de negócio do domínio

TheoryTab

musica_id (V1: 1–1 com música)

status ∈ {rascunho, em_revisao, publicado, arquivado}

Eventos do lifecycle estão bem claros no teu doc (NovaTabCriada, TabAprovada etc.).

TrilhaDeInstrumento

instrumento ∈ {piano, violao, guitarra}

tipo_conteudo e complexidade (simplificada/ completa)

ativo (bool)

BlocoTeorico

inicio_seg e duracao_seg (base da sincronização)

tipo: nota/acorde/texto

grau (1–7 opcional) e funcao_roman (I, IV, V...)

Controllers

Sugestão de superfície HTTP (sem amarrar em framework):

Público (Player)

GET /musicas/{id}/player?instrumento=piano → retorna PlayerTrackDTO

Admin (CRUD interno)

POST /admin/theory-tabs

PATCH /admin/theory-tabs/{id}/status

POST /admin/theory-tabs/{id}/trilhas

PUT /admin/trilhas/{id}/blocos

Services

Padrão obrigatório

service = caso de uso / orquestração

não conhece HTTP/SQL/framework

valida invariantes e delega persistência a repositories

Casos de uso

CreateTheoryTabService

SetTheoryTabStatusService

UpsertInstrumentTrackService

ReplaceBlocksBulkService

GetPlayerTrackService (read-model)

Repository e performance

Endpoints do player precisam buscar muitos blocos rápido e ordenado.

Query crítica: listBlocksByTrack(trilha_id) ORDER BY inicio_seg ASC

Índice obrigatório em (trilha_id, inicio_seg)

Estrutura de arquivos do módulo
src/modules/theory-tabs/
  controllers/
    player.controller.ts
    admin-theory-tabs.controller.ts
  services/
    create-theory-tab.service.ts
    set-theory-tab-status.service.ts
    upsert-instrument-track.service.ts
    replace-blocks-bulk.service.ts
    get-player-track.service.ts
  repository/
    theory-tab.repository.ts
    theory-tab.repository.db.ts
    instrument-track.repository.ts
    instrument-track.repository.db.ts
    theoretical-block.repository.ts
    theoretical-block.repository.db.ts
  entities/
    theory-tab.entity.ts
    instrument-track.entity.ts
    theoretical-block.entity.ts
  dtos/
    create-theory-tab.dto.ts
    set-status.dto.ts
    upsert-track.dto.ts
    replace-blocks.dto.ts
    player-track.dto.ts
  interfaces/
    theory-tab-repository.interface.ts
    instrument-track-repository.interface.ts
    theoretical-block-repository.interface.ts
  events/
    theory-tab-created.event.ts
    theory-tab-status-changed.event.ts
  index.ts
  README.md
