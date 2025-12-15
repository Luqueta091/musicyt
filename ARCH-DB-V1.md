Arquitetura do Banco v1.0

Documento ID: ARCH-DB-V1
Data: 2025-12-15
Status: Draft

A tua base de entidades já está excelente e eu vou formalizar com constraints/índices para V1.

ER Diagram
erDiagram
  USUARIO {
    uuid id PK
    string nome
    string email UK
    string senha_hash
    string tipo  "aluno|admin"
    datetime data_cadastro
  }

  MUSICA {
    uuid id PK
    string titulo
    string artista
    int duracao_segundos
    string tom_original
    string modo
    string youtube_id UK
    string youtube_url
    string nivel_dificuldade
    datetime criado_em
    datetime atualizado_em
  }

  THEORYTAB {
    uuid id PK
    uuid musica_id FK
    uuid criador_id FK
    string status "rascunho|em_revisao|publicado|arquivado"
    datetime data_criacao
    datetime data_publicacao
    string observacoes
  }

  TRILHA_INSTRUMENTO {
    uuid id PK
    uuid theorytab_id FK
    string instrumento "piano|violao|guitarra"
    string tipo_conteudo "melodia|acordes|melodia+acordes"
    string complexidade "simplificada|completa"
    boolean ativo
  }

  BLOCO_TEORICO {
    uuid id PK
    uuid trilha_id FK
    string tipo "nota|acorde|texto"
    int grau
    string funcao_roman
    string nota_letra
    int oitava
    float inicio_seg
    float duracao_seg
  }

  USUARIO ||--o{ THEORYTAB : cria
  MUSICA ||--|| THEORYTAB : possui
  THEORYTAB ||--o{ TRILHA_INSTRUMENTO : tem
  TRILHA_INSTRUMENTO ||--o{ BLOCO_TEORICO : compoe


Observação

Mantive MUSICA 1–1 THEORYTAB porque teu V1 indica isso (“link 1–1”).

Se no V2 você quiser múltiplas versões, a gente muda a constraint e adiciona versao.

Constraints e índices recomendados
Usuário

UK(email) — login simples

índice em (tipo) só se começar a filtrar admin/aluno frequentemente

Música

índice para busca:

(titulo) + (artista) (ou um índice composto / full-text dependendo do banco)

UK(youtube_id) (evitar duplicar música por URL)

TheoryTab

UK(musica_id) (garante 1–1 na V1)

índice em (status) pra listagem admin por status

Trilha

UK(theorytab_id, instrumento, complexidade, tipo_conteudo) (na V1 você pode simplificar pra UK(theorytab_id, instrumento))

Bloco

índice crítico: (trilha_id, inicio_seg)

check constraints:

inicio_seg >= 0

duracao_seg > 0

grau between 1 and 7 quando tipo=nota|acorde e aplicável

Estratégia de persistência

Banco relacional simples (tabelas normalizadas, FKs) — V1 vai ser tranquila.

cor não precisa persistir: é derivável por grau na UI (teu doc já sugere isso).

Checklist anti-gambiarra

Dois pontos que eu vou te segurar pela gola antes de virar dor:

Shared não é depósito de regra de negócio. Shared é infra e util genérico.

“Service em shared” só se for infra pura (email/logger/cache etc). Senão é violação clássica.
