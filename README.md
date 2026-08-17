# PixFlow

Sistema de transferências financeiras assíncronas inspirado no PIX, construído para demonstrar arquitetura de sistemas de pagamento distribuídos: processamento assíncrono via filas, consistência transacional, controle de concorrência e segurança de ponta a ponta.

---

## Índice

- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Fluxo de uma Transferência](#fluxo-de-uma-transferência)
- [Modelagem de Dados](#modelagem-de-dados)
- [Stack Tecnológica](#stack-tecnológica)
- [Segurança](#segurança)
- [Consistência e Concorrência](#consistência-e-concorrência)
- [Endpoints da API](#endpoints-da-api)
- [Como Rodar o Projeto](#como-rodar-o-projeto)
- [Estrutura de Pastas](#estrutura-de-pastas)
- [Testes](#testes)
- [Roadmap](#roadmap)

---

## Visão Geral

O **PixFlow** simula um sistema de transferência instantânea entre contas, processando as operações de forma **assíncrona** através de uma fila de mensagens. A API recebe a solicitação de transferência, valida os dados de entrada e delega o processamento financeiro a um **worker** dedicado, que garante — com lock transacional — que o saldo é validado e a operação é debitada/creditada de forma atômica, ou cancelada caso não haja saldo suficiente.

O objetivo do projeto é reproduzir, em escala reduzida, os desafios reais de um sistema de pagamentos:

- Processamento assíncrono e desacoplado
- Garantia de que dinheiro nunca é "perdido" ou "duplicado"
- Idempotência em operações financeiras
- Controle de concorrência em saldo compartilhado
- Autenticação, autorização e auditoria

---

## Arquitetura

```
                         ┌─────────────────────┐
                         │        Cliente        │
                         └──────────┬───────────┘
                                    │ HTTPS + JWT
                                    ▼
                    ┌───────────────────────────────┐
                    │      API REST (Spring Boot)     │
                    │  - Autenticação (JWT)           │
                    │  - Validação de entrada          │
                    │  - Verificação de idempotência   │
                    │  - Persiste Transação (PENDENTE) │
                    └───────────────┬───────────────┘
                                    │ publica evento
                                    ▼
                    ┌───────────────────────────────┐
                    │      Fila de Mensagens           │
                    │      (RabbitMQ / Kafka)          │
                    └───────────────┬───────────────┘
                                    │ consome evento
                                    ▼
                    ┌───────────────────────────────┐
                    │         Worker (Consumer)        │
                    │  - Lock nas contas envolvidas     │
                    │  - Valida saldo                  │
                    │  - Debita origem / Credita destino│
                    │  - Atualiza status da transação   │
                    │  - Publica resultado (opcional)   │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │          PostgreSQL               │
                    │  - Contas                        │
                    │  - Transações                    │
                    │  - Auditoria                     │
                    └───────────────────────────────┘
```

A separação entre **API (produtora)** e **Worker (consumidora)** permite que o sistema absorva picos de requisições sem travar o cliente: a API responde imediatamente com `202 Accepted`, e o processamento financeiro acontece em background, de forma controlada.

---

## Fluxo de uma Transferência

1. Cliente autenticado envia `POST /transferencias` com conta de origem, conta de destino, valor e uma `idempotencyKey` única.
2. API valida o payload e confere se o usuário autenticado é o dono da conta de origem.
3. API verifica se já existe uma transação com a mesma `idempotencyKey` — se sim, retorna o resultado já processado (evita duplicidade em caso de reenvio).
4. API persiste a transação no banco com status `PENDENTE` e publica um evento na fila.
5. API responde `202 Accepted` com o ID da transação, sem esperar o processamento.
6. Worker consome o evento da fila.
7. Worker aplica **lock pessimista** nas contas de origem e destino (sempre na mesma ordem, para evitar deadlock).
8. Worker valida se a conta de origem tem saldo suficiente.
   - **Se sim:** debita a origem, credita o destino, marca a transação como `CONCLUIDA`, tudo dentro de uma única transação de banco (`@Transactional`).
   - **Se não:** marca a transação como `CANCELADA`, sem alterar nenhum saldo.
9. Cliente consulta o status da transação via `GET /transferencias/{id}` (polling) ou recebe notificação (versão futura com WebSocket/Webhook).

Se o worker falhar de forma inesperada durante o processamento, a mensagem não é confirmada (`ack`) e volta para a fila, com limite de tentativas antes de ir para a **Dead Letter Queue (DLQ)** — evitando que uma falha transitória "perca" uma transferência.

---

## Modelagem de Dados

### Conta
| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Identificador único |
| titular | String | Nome do dono da conta |
| saldo | BigDecimal | Saldo atual |
| versao | Long | Controle de lock otimista (`@Version`) |
| criadoEm | Timestamp | Data de criação |

### Transacao
| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Identificador único |
| contaOrigemId | UUID | Conta que envia o valor |
| contaDestinoId | UUID | Conta que recebe o valor |
| valor | BigDecimal | Valor transferido |
| status | Enum | `PENDENTE`, `PROCESSANDO`, `CONCLUIDA`, `CANCELADA`, `FALHA` |
| idempotencyKey | String (único) | Chave para evitar duplicidade |
| motivoFalha | String | Preenchido quando cancelada/falhou |
| criadoEm | Timestamp | Data de criação |
| processadoEm | Timestamp | Data em que o worker finalizou |

### LogAuditoria
| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Identificador único |
| transacaoId | UUID | Referência à transação |
| evento | String | Ex: `TRANSACAO_CRIADA`, `SALDO_VALIDADO`, `DEBITO_REALIZADO` |
| detalhes | JSON | Dados adicionais do evento |
| criadoEm | Timestamp | Data do registro (imutável) |

---

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Linguagem | Java 21 |
| Framework | Spring Boot 3 |
| API | Spring Web (REST) |
| Persistência | Spring Data JPA + Hibernate |
| Banco de dados | PostgreSQL |
| Mensageria | RabbitMQ (Spring AMQP) |
| Autenticação | Spring Security + JWT |
| Rate Limiting | Bucket4j |
| Migrations | Flyway |
| Testes | JUnit 5, Mockito, Testcontainers |
| Containerização | Docker & Docker Compose |
| Documentação de API | springdoc-openapi (Swagger UI) |

---

## Segurança

O projeto trata a movimentação financeira como um domínio crítico, então diversas camadas de segurança foram aplicadas:

- **Autenticação via JWT** — todo endpoint de transferência exige um token válido.
- **Autorização por propriedade de recurso** — o usuário autenticado só pode iniciar transferências a partir de contas das quais é dono; qualquer tentativa de mover saldo de conta alheia é bloqueada na camada de API, antes mesmo de chegar à fila.
- **Idempotência obrigatória** — toda transferência exige uma `idempotencyKey` única, prevenindo débito duplicado em casos de retry de rede ou reenvio acidental do cliente.
- **Rate limiting por conta/usuário** — limita o número de tentativas de transferência em um intervalo de tempo, mitigando abuso e ataques de força bruta contra o sistema.
- **Validação estrita de entrada** — valores negativos, nulos, ou formatos inválidos são rejeitados na borda da API, nunca confiando em validação apenas no front-end.
- **Auditoria imutável** — cada etapa relevante do processamento (criação, validação de saldo, débito, crédito, cancelamento) gera um registro de auditoria, permitindo rastrear exatamente o que aconteceu com cada transação.
- **Least privilege no banco** — o usuário de banco utilizado pela aplicação possui apenas as permissões estritamente necessárias (sem DROP, sem acesso a schemas não utilizados).
- **Segredos fora do código** — chaves JWT, credenciais de banco e da fila são carregadas via variáveis de ambiente, nunca versionadas no repositório.

---

## Consistência e Concorrência

Movimentação de saldo é o ponto mais sensível do sistema. As garantias adotadas:

- **Lock pessimista nas contas envolvidas**, sempre adquirido na mesma ordem determinística (ex: por ID crescente), para eliminar risco de deadlock entre transferências concorrentes que envolvem as mesmas contas em sentidos opostos.
- **Transação atômica no worker** — débito, crédito e atualização de status da transação ocorrem dentro do mesmo escopo transacional (`@Transactional`); se qualquer etapa falhar, tudo é revertido (`rollback`).
- **Padrão Outbox (planejado)** — para garantir que a publicação do evento na fila só ocorra se a escrita no banco foi de fato confirmada, evitando cenários de mensagem publicada sem persistência correspondente (ou vice-versa).
- **Reprocessamento seguro** — como toda transação tem uma `idempotencyKey`, reprocessar a mesma mensagem da fila (em caso de reentrega) não gera efeito colateral duplicado.

---

## Endpoints da API

| Método | Rota | Descrição |
|---|---|---|
| POST | `/auth/login` | Autentica usuário e retorna JWT |
| POST | `/contas` | Cria uma nova conta |
| GET | `/contas/{id}` | Consulta dados e saldo de uma conta |
| POST | `/transferencias` | Solicita uma nova transferência (assíncrona) |
| GET | `/transferencias/{id}` | Consulta status de uma transferência |
| GET | `/contas/{id}/transferencias` | Lista histórico de transferências de uma conta |

Documentação interativa completa disponível via Swagger em `/swagger-ui.html` após subir o projeto.

---

## Como Rodar o Projeto

### Pré-requisitos
- Java 21+
- Docker e Docker Compose
- Maven

### Passos

```bash
# 1. Clone o repositório
git clone https://github.com/seu-usuario/pixflow.git
cd pixflow

# 2. Suba a infraestrutura (Postgres + RabbitMQ)
docker compose up -d

# 3. Rode as migrations e inicie a aplicação
./mvnw spring-boot:run
```

A API estará disponível em `http://localhost:8080` e o painel do RabbitMQ em `http://localhost:15672`.

---

## Estrutura de Pastas

```
pixflow/
├── src/
│   ├── main/
│   │   ├── java/com/pixflow/
│   │   │   ├── api/            # Controllers e DTOs
│   │   │   ├── config/         # Configurações (Security, Fila, Swagger)
│   │   │   ├── domain/         # Entidades e regras de negócio
│   │   │   ├── repository/     # Interfaces Spring Data JPA
│   │   │   ├── service/        # Regras de aplicação (API)
│   │   │   ├── worker/         # Consumers da fila e processamento financeiro
│   │   │   ├── security/       # JWT, filtros de autenticação
│   │   │   └── audit/          # Serviço de auditoria
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/   # Scripts Flyway
│   └── test/
│       └── java/com/pixflow/   # Testes unitários e de integração
├── docker-compose.yml
├── pom.xml
└── README.md
```

---

## Testes

- **Unitários** — regras de negócio isoladas (validação de saldo, cálculo, transições de status).
- **Integração** — fluxo completo API → fila → worker → banco, usando **Testcontainers** para subir Postgres e RabbitMQ reais em containers durante os testes.
- **Concorrência** — testes específicos simulando múltiplas transferências simultâneas envolvendo a mesma conta, validando que o saldo final está sempre correto.

```bash
./mvnw test
```

---

## Roadmap

- [ ] Implementação do padrão Outbox para publicação garantida de eventos
- [ ] Notificação em tempo real via WebSocket quando a transferência é concluída
- [ ] Suporte a Kafka como alternativa ao RabbitMQ
- [ ] Dashboard de monitoramento de transações (Grafana + Prometheus)
- [ ] Circuit breaker entre API e fila (Resilience4j)
- [ ] Suporte a estorno de transações concluídas

---

## Autor

Projeto desenvolvido como estudo de arquitetura de sistemas de pagamento distribuídos, aplicando conceitos de mensageria, consistência transacional e segurança em Java/Spring Boot.
