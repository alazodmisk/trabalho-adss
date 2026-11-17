# Plano de implementação — UC-003 · Inscrever-se em evento

> Escopo: implementar **somente** o UC-003 de ponta a ponta (Interface → API REST → Banco de dados).
> Tudo que não é necessário para o UC-003 funcionar e ser demonstrado fica **fora** (ver seção 1).
>
> **Duas opções para a API:** **Opção A — Node.js + Express** e **Opção B — Java + Spring Boot**.
> Banco, endpoints, erros, telas e testes são os mesmos nas duas; só mudam as ferramentas e o código das seções marcadas com **[A]** e **[B]**.
> Escolha uma e ignore a outra.

---

## 1. Escopo

### Dentro do escopo
- Participante identificado solicita inscrição em um evento.
- Sistema apresenta taxa e dados exigidos; participante informa/confirma.
- Validações: dados, vagas, período, evento cancelado, inscrição ativa duplicada.
- Evento **gratuito**: inscrição `CONFIRMADA` + QR Code disponível para consulta.
- Evento **pago**: inscrição `PENDENTE_PAGAMENTO` (sem QR Code) e aviso de que o pagamento segue no UC-004.
- Consulta da inscrição e do QR Code pelo próprio participante.

### Fora do escopo (somente *stubs* ou dados de seed)
| Item | Como será tratado |
|---|---|
| Cadastro/gestão de eventos | Eventos inseridos por `seed.sql` |
| Cadastro de usuários / recuperação de senha | Participantes inseridos por seed + login simples |
| Pagamento (UC-004) | Inscrição fica pendente; botão/rota de pagamento não implementados |
| Busca avançada, avaliação, cancelamento | Não implementados (apenas uma listagem simples de eventos) |
| Check-in / leitura do QR Code | Não implementado (só geração e exibição) |

---

## 2. Stack e ferramentas

### 2.1 Partes comuns às duas opções

| Camada | Escolha | Por quê |
|---|---|---|
| Banco | **PostgreSQL no Supabase** | Gratuito, hospedado (HTTPS pronto); transação + índice único parcial resolvem a concorrência (RNF-002) |
| Front end | **React + Vite** | Rápido de montar; HTML semântico facilita a acessibilidade (RNF-008) |
| QR Code | `qrcode.react` no front; a API devolve só um código opaco | Nenhum dado pessoal dentro do QR (RNF-004) |
| Autenticação | Login simples com **JWT** e senha com hash `bcrypt` | O UC exige "usuário identificado", mas auth não é o foco |

> O Supabase é usado **apenas como Postgres**: a API conecta pela *connection string*. Não usar `supabase-js` no front, para que nenhuma chave de banco chegue ao cliente.

### 2.2 Opção A — Node.js + Express

| Item | Ferramenta |
|---|---|
| Servidor | Express (TypeScript opcional) |
| Acesso ao banco | `pg` (SQL explícito, transação manual) |
| Validação | `zod` |
| JWT / senha | `jsonwebtoken` / `bcrypt` |
| Testes | Vitest ou Jest + Supertest |

### 2.3 Opção B — Java + Spring Boot (Java 17 ou 21)

| Item | Ferramenta |
|---|---|
| Servidor | Spring Boot 3 (Spring Web) — gerar em start.spring.io |
| Acesso ao banco | Spring Data JPA + driver PostgreSQL |
| Validação | Bean Validation (`@NotBlank`, `@Email`, `@Valid`) |
| JWT / senha | Spring Security + jjwt / `BCryptPasswordEncoder` |
| Testes | JUnit 5 + MockMvc / `TestRestTemplate` |

Dependências do `start.spring.io`: *Spring Web, Spring Data JPA, PostgreSQL Driver, Validation, Spring Security, Lombok (opcional)*. Adicionar o jjwt manualmente.

### 2.4 Comparação rápida

| Critério | A — Node | B — Java |
|---|---|---|
| Rapidez para montar | Maior (menos arquivos) | Menor (entidades, repositórios, DTOs, controllers) |
| Separação em camadas para o relatório | Boa | Muito explícita |
| Controle de transação e lock | SQL manual (`BEGIN`, `FOR UPDATE`) | Anotações (`@Transactional`, `@Lock`) |
| Deploy da API | Simples (build + start) | Exige `Dockerfile` |
| Recomendação | Se o grupo não tem preferência | Se o grupo já domina Java |

### 2.5 O que instalar na máquina

| Item | A — Node | B — Java |
|---|---|---|
| Node.js LTS + npm (para o front) | Sim | Sim |
| JDK 17 ou 21 | Não | Sim |
| Maven/Gradle | Não | Não precisa instalar: o projeto gerado traz o wrapper (`mvnw`/`gradlew`) |
| Editor / IDE | VS Code | IntelliJ IDEA Community, Eclipse ou VS Code com extensões Java |
| Git + conta GitHub | Sim | Sim |
| Conta no Supabase | Sim | Sim |
| Navegador moderno | Sim | Sim |

Opcionais (úteis para evidências): axe DevTools e Lighthouse (acessibilidade), Postman/Insomnia/`curl` (testar a API), leitor de tela (NVDA, VoiceOver ou Orca).
**Não é preciso** Docker, PostgreSQL local nem `psql`/DBeaver (o SQL Editor do Supabase basta); Docker só entra no deploy da opção B.

**Alternativa mais simples:** SQLite no lugar do Supabase (em qualquer das opções), tudo local. Funciona, mas a demonstração de concorrência (RNF-002) fica menos convincente.

---

## 3. Arquitetura

```
┌────────────────┐   HTTPS/JSON   ┌───────────────────────────┐   SQL    ┌─────────────────────┐
│  React (Vite)  │ ─────────────► │  API REST                 │ ───────► │ PostgreSQL/Supabase │
│  - Eventos     │ ◄───────────── │  (Express  ou  Spring)    │ ◄─────── │  eventos            │
│  - Inscrição   │                │  - auth (JWT)             │          │  participantes      │
│  - QR Code     │                │  - validação              │          │  inscricoes         │
└────────────────┘                │  - regras do UC-003       │          └─────────────────────┘
                                  └───────────────────────────┘
```

Camadas internas da API:
- **[A]** `routes` (HTTP) → `services` (regras do UC) → `db` (queries).
- **[B]** `controller` (HTTP) → `service` (regras do UC) → `repository` (JPA).

As regras de negócio ficam **apenas** na camada de service.

---

## 4. Modelo de dados

Mesmo SQL nas duas opções. Usei `text` + `check` no status (em vez de `enum` do Postgres) para funcionar sem configuração extra tanto no `pg` quanto no JPA.

```sql
create extension if not exists pgcrypto;

create table participantes (
  id           uuid primary key default gen_random_uuid(),
  nome         text not null,
  email        text not null unique,
  telefone     text,
  senha_hash   text not null,
  criado_em    timestamptz not null default now()
);

create table eventos (
  id                uuid primary key default gen_random_uuid(),
  titulo            text not null,
  descricao         text,
  local             text,
  data_evento       timestamptz not null,
  inscricoes_inicio timestamptz not null,
  inscricoes_fim    timestamptz not null,
  capacidade        integer not null check (capacidade > 0),
  taxa              numeric(10,2) not null default 0 check (taxa >= 0),
  cancelado         boolean not null default false,
  check (inscricoes_fim >= inscricoes_inicio)
);

create table inscricoes (
  id                uuid primary key default gen_random_uuid(),
  evento_id         uuid not null references eventos(id),
  participante_id   uuid not null references participantes(id),
  status            text not null
                    check (status in ('CONFIRMADA', 'PENDENTE_PAGAMENTO', 'CANCELADA')),
  valor_taxa        numeric(10,2) not null default 0,
  -- dados informados/confirmados no passo 3 do fluxo
  nome_inscrito     text not null,
  email_inscrito    text not null,
  telefone_inscrito text,
  codigo_qr         text unique,           -- só preenchido quando CONFIRMADA
  criada_em         timestamptz not null default now()
);

-- Garante unicidade mesmo sob concorrência (RF-005 / RNF-002):
create unique index uq_inscricao_ativa
  on inscricoes (evento_id, participante_id)
  where status in ('CONFIRMADA', 'PENDENTE_PAGAMENTO');

create index idx_inscricoes_evento_status on inscricoes (evento_id, status);

-- Segurança (RNF-004): o Supabase expõe tabelas via API pública.
-- Ativar RLS SEM criar policies bloqueia o acesso direto por chave anon;
-- somente a API (conexão direta) acessa os dados.
alter table participantes enable row level security;
alter table eventos       enable row level security;
alter table inscricoes    enable row level security;
```

**Vagas ocupadas** = número de inscrições com status `CONFIRMADA` ou `PENDENTE_PAGAMENTO` no evento.
`vagas_restantes = capacidade − vagas ocupadas` (calculado, nunca armazenado → evita inconsistência).

---

## 5. API REST

Base: `/api` · JSON · `Authorization: Bearer <jwt>` em tudo, exceto login.
O `participante_id` **sempre vem do token**, nunca do corpo da requisição.

| Método | Rota | Descrição | Sucesso |
|---|---|---|---|
| POST | `/auth/login` | Autentica e devolve JWT *(apoio)* | 200 |
| GET | `/eventos` | Lista eventos com `vagas_restantes`, `taxa`, `inscricoes_abertas` *(apoio)* | 200 |
| GET | `/eventos/:id` | Detalhe do evento + **dados exigidos** + taxa (passo 2 do fluxo) | 200 |
| **POST** | **`/eventos/:id/inscricoes`** | **Solicita a inscrição (núcleo do UC-003)** | **201** |
| GET | `/inscricoes/me` | Inscrições do participante logado | 200 |
| GET | `/inscricoes/:id` | Detalhe de uma inscrição (somente do dono) | 200 |
| GET | `/inscricoes/:id/qrcode` | Código do QR (somente do dono e se `CONFIRMADA`) | 200 |

### `POST /eventos/:id/inscricoes`

Requisição:
```json
{ "nome": "Maria Souza", "email": "maria@exemplo.com", "telefone": "(44) 99999-0000" }
```

Resposta 201 (evento gratuito):
```json
{
  "id": "…",
  "evento_id": "…",
  "status": "CONFIRMADA",
  "valor_taxa": 0,
  "qrcode_disponivel": true
}
```

Resposta 201 (evento pago):
```json
{
  "id": "…",
  "status": "PENDENTE_PAGAMENTO",
  "valor_taxa": 30.00,
  "qrcode_disponivel": false,
  "proximo_passo": "PAGAMENTO"
}
```

### Erros (formato único)
```json
{ "erro": { "codigo": "EVENTO_LOTADO", "mensagem": "Não há mais vagas para este evento.", "campos": {} } }
```

| Situação (exceção do UC) | HTTP | `codigo` |
|---|---|---|
| Dados inválidos / ausentes | 422 | `DADOS_INVALIDOS` (com `campos` indicando cada erro) |
| Evento inexistente | 404 | `EVENTO_NAO_ENCONTRADO` |
| Evento cancelado | 409 | `EVENTO_CANCELADO` |
| Fora do período de inscrição | 409 | `FORA_DO_PERIODO` |
| Evento lotado | 409 | `EVENTO_LOTADO` |
| Inscrição ativa duplicada | 409 | `INSCRICAO_DUPLICADA` |
| Sem token / token inválido | 401 | `NAO_AUTENTICADO` |
| Inscrição de outro participante | 404 | `INSCRICAO_NAO_ENCONTRADA` (404, e não 403, para não revelar a existência) |

---

## 6. Lógica da inscrição (coração do UC-003)

### 6.1 Algoritmo (igual nas duas opções)

1. **Validar o formato dos dados** (nome, e-mail, telefone) → 422 `DADOS_INVALIDOS`.
2. **Abrir transação** e **travar a linha do evento** (`SELECT ... FOR UPDATE`). Isso faz com que inscrições simultâneas do *mesmo* evento sejam processadas uma de cada vez (RNF-002).
3. Evento não existe → 404. Cancelado → 409. Fora do período → 409.
4. Já existe inscrição ativa (`CONFIRMADA` ou `PENDENTE_PAGAMENTO`) do participante nesse evento → 409 `INSCRICAO_DUPLICADA`.
5. Vagas ocupadas ≥ capacidade → 409 `EVENTO_LOTADO`.
6. Taxa = 0 → status `CONFIRMADA` e gera `codigo_qr` aleatório e único (RF-008). Taxa > 0 → status `PENDENTE_PAGAMENTO`, sem QR (segue para o UC-004).
7. Inserir a inscrição e **confirmar a transação**.
8. **Rede de segurança:** se o índice único parcial for violado, devolver 409 `INSCRICAO_DUPLICADA`.

**Dupla proteção contra duplicidade:** verificação dentro da transação (mensagem amigável) + índice único parcial no banco (garantia final).

### 6.2 [A] Implementação em Node.js

```ts
// services/inscricaoService.ts  (pseudocódigo)
async function inscrever(participanteId, eventoId, dados) {
  validarDados(dados);                                  // zod → 422

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const ev = await client.query(
      'SELECT * FROM eventos WHERE id = $1 FOR UPDATE', [eventoId]);
    if (!ev.rows[0])            throw erro(404, 'EVENTO_NAO_ENCONTRADO');
    const evento = ev.rows[0];

    if (evento.cancelado)                                   throw erro(409, 'EVENTO_CANCELADO');
    if (agora < evento.inscricoes_inicio ||
        agora > evento.inscricoes_fim)                      throw erro(409, 'FORA_DO_PERIODO');

    const dup = await client.query(
      `SELECT 1 FROM inscricoes
        WHERE evento_id=$1 AND participante_id=$2
          AND status IN ('CONFIRMADA','PENDENTE_PAGAMENTO')`, [eventoId, participanteId]);
    if (dup.rowCount)                                       throw erro(409, 'INSCRICAO_DUPLICADA');

    const ocup = await client.query(
      `SELECT count(*)::int AS n FROM inscricoes
        WHERE evento_id=$1 AND status IN ('CONFIRMADA','PENDENTE_PAGAMENTO')`, [eventoId]);
    if (ocup.rows[0].n >= evento.capacidade)                throw erro(409, 'EVENTO_LOTADO');

    const gratuito = Number(evento.taxa) === 0;
    const status   = gratuito ? 'CONFIRMADA' : 'PENDENTE_PAGAMENTO';
    const codigoQr = gratuito ? crypto.randomBytes(16).toString('hex') : null;

    const ins = await client.query(/* INSERT INTO inscricoes ... RETURNING * */);
    await client.query('COMMIT');
    return ins.rows[0];
  } catch (e) {
    await client.query('ROLLBACK');
    if (e.code === '23505') throw erro(409, 'INSCRICAO_DUPLICADA');   // índice único
    throw e;
  } finally {
    client.release();
  }
}
```

### 6.3 [B] Implementação em Java (Spring Boot)

**DTO com validação**
```java
public record InscricaoRequest(
    @NotBlank @Size(max = 120) String nome,
    @NotBlank @Email           String email,
    @Pattern(regexp = "^[0-9()+\\-\\s]{8,20}$") String telefone) {}
```

**Repositórios (Spring Data JPA)**
```java
public interface EventoRepository extends JpaRepository<Evento, UUID> {
    // SELECT ... FOR UPDATE: serializa inscrições do mesmo evento (RNF-002)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select e from Evento e where e.id = :id")
    Optional<Evento> findByIdForUpdate(@Param("id") UUID id);
}

public interface InscricaoRepository extends JpaRepository<Inscricao, UUID> {
    boolean existsByEventoIdAndParticipanteIdAndStatusIn(
        UUID eventoId, UUID participanteId, Collection<StatusInscricao> status);

    long countByEventoIdAndStatusIn(UUID eventoId, Collection<StatusInscricao> status);
}
```

**Service (regras do UC-003)**
```java
@Service
public class InscricaoService {

    private static final List<StatusInscricao> ATIVAS =
        List.of(StatusInscricao.CONFIRMADA, StatusInscricao.PENDENTE_PAGAMENTO);

    private final EventoRepository eventos;
    private final InscricaoRepository inscricoes;
    private final SecureRandom random = new SecureRandom();

    // construtor com injeção omitido

    @Transactional                                   // BEGIN ... COMMIT/ROLLBACK
    public Inscricao inscrever(UUID participanteId, UUID eventoId, InscricaoRequest d) {

        Evento ev = eventos.findByIdForUpdate(eventoId)
            .orElseThrow(() -> new NegocioException(404, "EVENTO_NAO_ENCONTRADO"));

        if (ev.isCancelado())
            throw new NegocioException(409, "EVENTO_CANCELADO");

        Instant agora = Instant.now();
        if (agora.isBefore(ev.getInscricoesInicio()) || agora.isAfter(ev.getInscricoesFim()))
            throw new NegocioException(409, "FORA_DO_PERIODO");

        if (inscricoes.existsByEventoIdAndParticipanteIdAndStatusIn(eventoId, participanteId, ATIVAS))
            throw new NegocioException(409, "INSCRICAO_DUPLICADA");

        if (inscricoes.countByEventoIdAndStatusIn(eventoId, ATIVAS) >= ev.getCapacidade())
            throw new NegocioException(409, "EVENTO_LOTADO");

        boolean gratuito = ev.getTaxa().signum() == 0;

        Inscricao i = new Inscricao();
        i.setEventoId(eventoId);
        i.setParticipanteId(participanteId);
        i.setNomeInscrito(d.nome());
        i.setEmailInscrito(d.email());
        i.setTelefoneInscrito(d.telefone());
        i.setValorTaxa(ev.getTaxa());
        i.setStatus(gratuito ? StatusInscricao.CONFIRMADA : StatusInscricao.PENDENTE_PAGAMENTO);
        if (gratuito) i.setCodigoQr(gerarCodigo());          // RF-008

        return inscricoes.saveAndFlush(i);   // flush força a checagem do índice único aqui
    }

    private String gerarCodigo() {
        byte[] bytes = new byte[16];
        random.nextBytes(bytes);
        return HexFormat.of().formatHex(bytes);
    }
}
```

**Tratamento global de erros (formato único da seção 5)**
```java
@RestControllerAdvice
public class ErrosHandler {

    @ExceptionHandler(NegocioException.class)           // 404 / 409 do UC
    ResponseEntity<ErroResponse> negocio(NegocioException e) { ... }

    @ExceptionHandler(MethodArgumentNotValidException.class)   // @Valid falhou → 422
    ResponseEntity<ErroResponse> validacao(MethodArgumentNotValidException e) { ... }

    @ExceptionHandler(DataIntegrityViolationException.class)   // índice único → 409
    ResponseEntity<ErroResponse> integridade(DataIntegrityViolationException e) {
        return resposta(409, "INSCRICAO_DUPLICADA");
    }
}
```

**Configuração (`application.properties`)**
```properties
spring.datasource.url=${DATABASE_URL}          # jdbc:postgresql://host:5432/postgres?sslmode=require
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=validate         # o schema vem do schema.sql, não do Hibernate
spring.jpa.open-in-view=false
app.jwt.secret=${JWT_SECRET}
```

**Pontos de atenção na opção B**
- Se usar o *pooler* do Supabase em modo transação, adicione `prepareThreshold=0` à URL JDBC (ou use a conexão direta / pooler em modo sessão). No pooler, o usuário tem o formato `postgres.<ref-do-projeto>`.
- Mapeie `status` como `@Enumerated(EnumType.STRING)`; funciona com a coluna `text` do schema.
- `ddl-auto=validate` evita que o Hibernate altere as tabelas e dispense o índice único parcial, que ele não sabe criar.
- Lombok é opcional; sem ele, gere getters/setters pela IDE.

---

## 7. Interface gráfica

Igual nas duas opções. Telas mínimas (4):

| Tela | Conteúdo |
|---|---|
| **Login** | E-mail e senha *(apoio)* |
| **Eventos** | Lista simples: título, data, taxa, vagas restantes, botão "Ver / Inscrever-se" |
| **Inscrição** (`/eventos/:id/inscricao`) | Resumo do evento + **taxa** + formulário pré-preenchido com **dados exigidos** (passos 2–3); botão "Confirmar inscrição" |
| **Minha inscrição** (`/inscricoes/:id`) | Status; se `CONFIRMADA`, exibe QR Code (e o código em texto); se `PENDENTE_PAGAMENTO`, aviso "aguardando pagamento (UC-004)" |

### Estados da tela de Inscrição
`carregando → formulário → enviando → sucesso (redireciona p/ Minha inscrição) | erro`
Cada código de erro da seção 5 tem uma mensagem própria em português.

### Acessibilidade (RNF-008) — checklist de implementação
- [ ] Todos os campos com `<label for>` (nada de placeholder como rótulo).
- [ ] Fluxo completo usável **só com teclado** (Tab, Shift+Tab, Enter); ordem de foco lógica.
- [ ] Foco visível: **não** remover `outline`; estilizar `:focus-visible` com contraste claro.
- [ ] Erros comunicados por **texto + ícone** (nunca só cor vermelha), ligados ao campo via `aria-describedby` e `aria-invalid`.
- [ ] Resumo de erros com `role="alert"` / `aria-live`, e foco movido para ele após submit com falha.
- [ ] QR Code com `alt`/`aria-label` descritivo e o código também em texto.
- [ ] Botões como `<button>`, links como `<a>`; contraste mínimo AA.
- [ ] Validar com **Lighthouse** e **axe DevTools** e guardar os prints para o relatório.

---

## 8. Segurança e privacidade

| Requisito | Medida |
|---|---|
| RNF-004 Privacidade | API devolve só dados do próprio participante; QR Code contém apenas código aleatório; e-mail de outros nunca aparece em respostas; não logar corpo das requisições com dados pessoais; RLS ativado nas tabelas |
| RNF-005 Comunicações | Front e API publicados em HTTPS; credenciais do banco e `JWT_SECRET` só em variáveis de ambiente do servidor; front só conhece a URL da API (`VITE_API_URL`); CORS restrito ao domínio do front; `.env` / `application-local.properties` no `.gitignore` (**nunca** commitar a connection string) |
| Geral | Senha com hash (**[A]** `bcrypt`, **[B]** `BCryptPasswordEncoder`); consultas parametrizadas (**[A]** `$1, $2`, **[B]** JPA/parâmetros nomeados) contra SQL injection; **[A]** `helmet`, **[B]** cabeçalhos padrão do Spring Security |

---

## 9. Rastreabilidade (para o relatório)

### Requisitos funcionais
| Requisito | Onde é atendido | Como demonstrar |
|---|---|---|
| **RF-004** Solicitar inscrição | `POST /eventos/:id/inscricoes` + tela Inscrição + tabela `inscricoes` | Inscrever-se em evento gratuito e ver o registro no banco |
| **RF-005** Validar vagas e duplicidade | Verificações na transação + índice `uq_inscricao_ativa` | Tentar inscrever-se duas vezes; tentar em evento lotado |
| **RF-008** Disponibilizar QR Code | `codigo_qr` único + `GET /inscricoes/:id/qrcode` + tela Minha inscrição | Abrir a inscrição confirmada e ver o QR |

### Requisitos não funcionais
| Requisito | Onde é atendido | Como demonstrar |
|---|---|---|
| **RNF-002** Integridade | Lock na linha do evento (`FOR UPDATE`) + índice único parcial | Teste de concorrência (seção 11) |
| **RNF-004** Privacidade | Autorização por dono, RLS, QR opaco | Tentar acessar inscrição de outro usuário → 404; tentar ler tabela com chave anon → negado |
| **RNF-005** Segurança das comunicações | HTTPS no deploy; segredos só no servidor | Print do cadeado/URL `https://`; inspecionar o bundle do front sem segredos |
| **RNF-008** Acessibilidade | Checklist da seção 7 | Vídeo/prints navegando só por teclado; relatório do axe/Lighthouse |

### Regras de negócio
O texto das regras **RN-001, RN-003, RN-004, RN-005, RN-006, RN-007, RN-009 e RN-017** não estava nos arquivos usados para montar este plano. Copie cada uma do relatório e mapeie para um dos passos do algoritmo da seção 6.1 (dados, evento cancelado, período, duplicidade, vagas, gratuito × pago):

| RN | Texto da regra (copiar do relatório) | Onde é aplicada | Teste que cobre |
|---|---|---|---|
| RN-001 | | | |
| RN-003 | | | |
| RN-004 | | | |
| RN-005 | | | |
| RN-006 | | | |
| RN-007 | | | |
| RN-009 | | | |
| RN-017 | | | |

---

## 10. Estrutura de pastas sugerida

### [A] Node.js
```
trabalho-adss/
├── docs/                      # relatório, diagramas, prints de evidência
├── backend/
│   ├── sql/
│   │   ├── schema.sql
│   │   └── seed.sql
│   ├── src/
│   │   ├── routes/            # auth.ts, eventos.ts, inscricoes.ts
│   │   ├── services/          # inscricaoService.ts  ← regras do UC-003
│   │   ├── middlewares/       # auth (JWT), errorHandler
│   │   ├── db/                # pool.ts
│   │   └── server.ts
│   ├── tests/                 # testes da API + concorrência
│   └── .env.example
└── frontend/
    └── src/
        ├── pages/             # Login, Eventos, Inscricao, MinhaInscricao
        ├── components/        # Campo de formulário acessível, ResumoErros, QrCode
        └── api/               # cliente HTTP
```

### [B] Java + Spring Boot
```
trabalho-adss/
├── docs/
├── backend/
│   ├── sql/
│   │   ├── schema.sql
│   │   └── seed.sql
│   ├── src/main/java/br/edu/adss/uc003/
│   │   ├── controller/        # AuthController, EventoController, InscricaoController
│   │   ├── service/           # InscricaoService  ← regras do UC-003
│   │   ├── repository/        # EventoRepository, InscricaoRepository, ParticipanteRepository
│   │   ├── model/             # Evento, Inscricao, Participante, StatusInscricao
│   │   ├── dto/               # InscricaoRequest, InscricaoResponse, ErroResponse
│   │   ├── exception/         # NegocioException, ErrosHandler
│   │   ├── security/          # filtro JWT, SecurityConfig
│   │   └── Uc003Application.java
│   ├── src/main/resources/application.properties
│   ├── src/test/java/...      # testes da API + concorrência
│   ├── pom.xml
│   └── Dockerfile             # só para o deploy
└── frontend/                  # igual à opção A
```

### Seed recomendado (nas duas opções)
Cobre todos os caminhos do UC sem precisar mexer no banco durante a demo:

| Evento | Característica | Cenário que demonstra |
|---|---|---|
| A | Gratuito, aberto, com vagas | Fluxo principal + QR Code |
| B | Pago, aberto, com vagas | Inscrição pendente → UC-004 |
| C | Gratuito, capacidade 1 já ocupada | Exceção: lotado |
| D | `cancelado = true` | Exceção: cancelado |
| E | Período de inscrição encerrado | Exceção: fora do período |

Participantes de teste: pelo menos 3 (um para o fluxo principal, um para duplicidade e um para testar acesso à inscrição de outro).

---

## 11. Plano de testes

### 11.1 Testes de API (iguais nas duas opções)
Ferramenta: **[A]** Vitest/Jest + Supertest · **[B]** JUnit 5 + MockMvc/`TestRestTemplate` · ou uma coleção Postman/Insomnia.

| # | Cenário | Esperado |
|---|---|---|
| 1 | Inscrição em evento gratuito | 201, `CONFIRMADA`, `codigo_qr` preenchido |
| 2 | Inscrição em evento pago | 201, `PENDENTE_PAGAMENTO`, sem `codigo_qr` |
| 3 | Dados inválidos (e-mail mal formado, nome vazio) | 422 com `campos` |
| 4 | Mesma inscrição duas vezes | 2ª → 409 `INSCRICAO_DUPLICADA` |
| 5 | Evento lotado | 409 `EVENTO_LOTADO` |
| 6 | Evento cancelado | 409 `EVENTO_CANCELADO` |
| 7 | Fora do período | 409 `FORA_DO_PERIODO` |
| 8 | Sem token | 401 |
| 9 | Ler inscrição de outro participante | 404 |
| 10 | QR Code de inscrição pendente | Negado (409/404) |

### 11.2 Teste de concorrência (RNF-002) — o mais valioso para a nota
Dispare requisições **em paralelo**:
- **20 participantes diferentes** no evento com capacidade **5** → exatamente **5** × 201 e **15** × 409 `EVENTO_LOTADO`; no banco, 5 linhas.
- **1 participante** enviando **10 requisições simultâneas** ao mesmo evento → exatamente **1** × 201 e 9 × 409 `INSCRICAO_DUPLICADA`.

**[A] Node:** script com `Promise.all` de vários `fetch`.

**[B] Java:** `ExecutorService` com um `CountDownLatch` para largar todas as threads ao mesmo tempo.
```java
ExecutorService pool = Executors.newFixedThreadPool(20);
CountDownLatch largada = new CountDownLatch(1);
List<Future<Integer>> resultados = new ArrayList<>();

for (String token : tokensDosParticipantes) {
    resultados.add(pool.submit(() -> {
        largada.await();                       // todas esperam o mesmo sinal
        return postInscricao(eventoId, token); // devolve o status HTTP
    }));
}
largada.countDown();                           // largada!

long criadas = contar(resultados, 201);        // esperado: 5
long lotadas = contar(resultados, 409);        // esperado: 15
```

### 11.3 Testes manuais de interface
Fluxo principal e cada mensagem de erro; navegação somente por teclado; leitor de tela (NVDA/VoiceOver) em ao menos a tela de Inscrição; teste em tela estreita (celular).

---

## 12. Etapas de implementação (ordem sugerida)

| Etapa | Entrega | Esforço |
|---|---|---|
| 0. Preparação | Projeto no Supabase; pastas `backend/` e `frontend/`; **[A]** `npm init` + Express · **[B]** projeto gerado no start.spring.io; `.env.example`, `.gitignore` | P |
| 1. Banco | `schema.sql` + `seed.sql` executados no Supabase | P |
| 2. API base | Servidor, conexão com o banco, login/JWT, `GET /eventos` e `GET /eventos/:id` | M |
| 3. **Núcleo do UC** | `POST /eventos/:id/inscricoes` com transação e todos os erros | M |
| 4. Consulta | `GET /inscricoes/me`, `/:id`, `/:id/qrcode` | P |
| 5. Testes da API | Testes 1–10 + teste de concorrência | M |
| 6. Front end | Login, Eventos, Inscrição, Minha inscrição (já acessíveis desde o início) | M |
| 7. Acessibilidade | Passar o checklist + axe/Lighthouse e corrigir | P |
| 8. Deploy | Banco (Supabase) + API + front, tudo em HTTPS; variáveis de ambiente configuradas. **[A]** Render/Railway direto do repositório · **[B]** Render/Railway com `Dockerfile` | P/M |
| 9. Evidências e relatório | Prints, vídeo curto da demo, tabela de rastreabilidade preenchida | P |

*(P = pequeno, M = médio.)* Se o tempo apertar, a **ordem de corte** é: deploy hospedado (rodar local e documentar) → leitor de tela → tela de login (usar usuário fixo por variável de ambiente).

**[B] Dockerfile mínimo para o deploy da API em Java**
```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
CMD ["java", "-jar", "app.jar"]
```

---

## 13. Decisões em aberto (alinhar com o relatório antes de codar)

1. **Linguagem da API:** A (Node) ou B (Java)? Decidir antes da etapa 0; o resto do plano não muda.
2. **Inscrição pendente ocupa vaga?** Este plano assume **sim** (reserva a vaga enquanto aguarda o pagamento). Confirme com as RNs; se a regra for outra, ajuste a contagem de vagas ocupadas.
3. **Quais são os "dados exigidos"?** O plano usa nome, e-mail e telefone. Troque pelos dados definidos no relatório.
4. **O que significa "usuário identificado"?** Aqui: login com JWT. Se o relatório definir outro mecanismo, adapte apenas a camada de autenticação.
5. **Prazo de pendência de pagamento:** não tratado (pertence ao UC-004). Se uma RN falar em expiração da inscrição pendente, será necessário um campo `expira_em`.

---

## 14. Checklist final de entrega

- [ ] Fluxo principal funcionando de ponta a ponta (UI → API → banco).
- [ ] As exceções do UC demonstráveis (dados inválidos, duplicada, lotado, cancelado, fora do período).
- [ ] Evento pago termina em `PENDENTE_PAGAMENTO` e sinaliza o UC-004.
- [ ] Teste de concorrência passando.
- [ ] Nenhum segredo no repositório; app publicado em HTTPS (ou justificativa documentada).
- [ ] Relatório de acessibilidade (axe/Lighthouse) anexado.
- [ ] Tabela de rastreabilidade RF/RNF/RN preenchida.
- [ ] README com instruções para rodar (`.env`, `schema.sql`, `seed.sql`, comandos da opção escolhida).
