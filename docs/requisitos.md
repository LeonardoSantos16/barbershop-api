# Requisitos

> RF = requisito funcional · RNF = requisito não funcional · RN = ver `regras-de-negocio.md`.
> Prioridade: **Núcleo** (MVP obrigatório) · **Extra** (MVP, se houver tempo) · **Futuro**.

## Requisitos funcionais

### Cliente (público, sem autenticação)

| ID | Requisito | Regras | Prioridade |
|---|---|---|---|
| RF01 | Listar serviços ativos com nome, duração e preço | RN15, RN16 | Núcleo |
| RF02 | Consultar horários de início disponíveis de um serviço em uma data | RN04–RN09, RN11, RN12 | Núcleo |
| RF03 | Criar um agendamento informando serviço, início, nome, telefone e e-mail opcional; a resposta devolve o token de cancelamento | RN01–RN10 | Núcleo |
| RF04 | Consultar um agendamento pelo token de cancelamento (dados para a tela de cancelamento) | RN17 | Núcleo |
| RF05 | Cancelar um agendamento pelo token | RN17, RN19 | Núcleo |
| RF06 | Guardar o token no localStorage ao agendar e listar os "meus próximos agendamentos" ativos daquele navegador | RN17 | Extra |

### Barbeiro/Admin (autenticado)

| ID | Requisito | Regras | Prioridade |
|---|---|---|---|
| RF07 | Fazer login com e-mail e senha | RN25 | Núcleo |
| RF08 | Criar, editar, desativar e listar serviços (incluindo inativos) | RN15, RN16 | Núcleo |
| RF09 | Definir e consultar o horário semanal (períodos por dia da semana) | RN11, RN13 | Núcleo |
| RF10 | Criar, editar, remover e listar ajustes por data, incluindo folgas | RN12, RN13 | Núcleo |
| RF11 | Consultar e alterar configurações: intervalo, antecedência mínima, antecedência máxima, limite por telefone | RN14 | Núcleo |
| RF12 | Ver a agenda (agendamentos por período, com dados do cliente e do serviço) | — | Núcleo |
| RF13 | Cancelar um agendamento | RN18, RN19 | Núcleo |
| RF14 | Receber no painel aberto um aviso de novo agendamento | RN23 | Extra |

### Sistema

| ID | Requisito | Regras | Prioridade |
|---|---|---|---|
| RF15 | Enviar e-mail de confirmação com link de cancelamento | RN21 | Núcleo |
| RF16 | Enviar e-mail de aviso quando o barbeiro cancela | RN21 | Núcleo |
| RF17 | Enviar lembretes por e-mail a partir de um job disparado periodicamente por um agendador | RN22 | Extra |

### Futuro (registrado, não implementar)

| ID | Requisito |
|---|---|
| RF-F01 | Histórico de agendamentos do cliente |
| RF-F02 | Conta e login do cliente |
| RF-F03 | Múltiplos barbeiros e separação de papéis dono × barbeiro |
| RF-F04 | Multi-tenant (várias barbearias) |
| RF-F05 | Barbeiro agendar em nome do cliente (encaixe) |
| RF-F06 | Status "concluído" e "não compareceu" |
| RF-F07 | Notificações por WhatsApp (Cloud API) |
| RF-F08 | Web Push |
| RF-F09 | Recuperação de senha |
| RF-F10 | Pagamento/sinal |

## Requisitos não funcionais

### Contrato e API

| ID | Requisito |
|---|---|
| RNF01 | **Contract-first**: o contrato OpenAPI 3.1 (YAML, escrito à mão) é criado ou alterado **antes** do código de cada fatia e versionado no repositório da API. |
| RNF02 | O contrato passa no lint do **Spectral** no CI; um contrato inválido quebra o build. |
| RNF03 | A API (Fastify) valida requisições e respostas **a partir do contrato**, sem schemas duplicados. |
| RNF04 | O front consome tipos e client gerados do contrato (`openapi-typescript` + `openapi-fetch`) e pode rodar contra o mock do **Prism**. |
| RNF05 | Erros em formato padronizado (RFC 9457 *Problem Details*), com códigos de erro estáveis em inglês (ex.: `SLOT_UNAVAILABLE`, `SCHEDULE_CONFLICT`). |
| RNF06 | Rotas, campos e códigos em inglês; recursos no plural; API versionada no caminho (`/v1`). |
| RNF07 | *(Extra)* Testes de contrato com **Schemathesis** no CI contra a API em execução. |

### Testes e qualidade

| ID | Requisito |
|---|---|
| RNF08 | Testes unitários do domínio (disponibilidade, agendamento, horários, cancelamento) com **100% de linhas e branches**. |
| RNF09 | Cobertura global mínima de **~80%**, exigida no CI. |
| RNF10 | Testes de integração contra **Postgres real** (Testcontainers). |
| RNF11 | Teste de concorrência: N requisições simultâneas para o mesmo horário resultam em exatamente 1 agendamento (garantido no banco, ex.: exclusion constraint com `tstzrange`). **Primeiro item a cortar** se o prazo apertar. |
| RNF12 | Lint e checagem de tipos (TypeScript strict) no CI, nos dois repositórios. |

### Segurança

| ID | Requisito |
|---|---|
| RNF13 | Senha do admin armazenada com hash forte (argon2 ou bcrypt); autenticação via JWT com expiração. |
| RNF14 | Token de cancelamento aleatório (≥ 128 bits), armazenado **como hash** no banco. |
| RNF15 | Rate limit por IP no endpoint de criação de agendamento (limite configurável por variável de ambiente), com contador compartilhado no ElastiCache. |
| RNF16 | Endpoint interno de jobs protegido por segredo compartilhado (header). |
| RNF17 | Segredos só em variáveis de ambiente; nada sensível no repositório. |
| RNF18 | CORS restrito ao domínio do front. |
| RNF19 | RDS e ElastiCache em subnets privadas, acessíveis **somente** a partir do security group da API (ver ADR 0001). |

### Arquitetura

| ID | Requisito |
|---|---|
| RNF20 | Domínio isolado de infraestrutura (regras testáveis sem banco e sem HTTP). |
| RNF21 | Envio de e-mail atrás de uma interface (`EmailSender`); implementação via **Gmail API (HTTPS)**. |
| RNF22 | Preparação para multi-tenant conforme ADR (ver `visao.md` §7). |
| RNF23 | Datas armazenadas em UTC; regras avaliadas em America/Sao_Paulo. |
| RNF24 | Migrations versionadas e seed reprodutível (admin, serviços iniciais e horário padrão). |
| RNF25 | Decisões relevantes registradas como ADR; cada fatia especificada como *change* no OpenSpec. |
| RNF32 | E-mails enviados de forma assíncrona: a API publica na fila SQS e o worker (Lambda, fora da VPC) consome e envia. A mensagem carrega todos os dados do e-mail; falhas repetidas vão para uma DLQ. |
| RNF33 | Cache (ElastiCache) **somente** na consulta de "meus agendamentos" do cliente, invalidado quando o cliente faz uma requisição que não é GET. Endpoints do admin e de horários disponíveis **nunca** são cacheados. |

### Operação e deploy

| ID | Requisito |
|---|---|
| RNF26 | **Infraestrutura AWS no emulador oficial**: ECS Fargate + ALB (API), Lambda (worker de e-mail), RDS PostgreSQL, ElastiCache, SQS e CloudWatch (ver ADR 0001). Gmail API (e-mail) e GitHub Actions (CI). |
| RNF27 | CI no GitHub Actions em todo PR: lint do contrato, lint, tipos, testes e cobertura. Estratégia de deploy no ambiente AWS a definir. |
| RNF28 | Endpoint `/health` (API e conexão com o banco), usado pelo health check do ALB e do ECS. |
| RNF29 | Logs estruturados (pino) com id de requisição, enviados ao CloudWatch Logs. |
| RNF30 | A API mantém uma task Fargate sempre em execução, sem *cold start*. O *cold start* é aceito apenas no worker (Lambda), que é assíncrono. |
| RNF31 | Um agendador dispara o job de lembretes periodicamente (ex.: a cada 10 min; mecanismo a definir em ADR); o processamento é idempotente e tolera atraso e falhas. |

## Riscos conhecidos

| Risco | Mitigação |
|---|---|
| O emulador não suportar ou não reproduzir fielmente algum serviço (ex.: ECS, ALB, SQS→Lambda) | Verificar a cobertura do emulador antes de adotar cada serviço; registrar divergências em ADR |
| Custo em uma conta AWS real (~US$ 64/mês pela estimativa do ADR 0001) | Executar no emulador; o custo é aceito apenas se houver deploy real |
| API com uma única task, sem alta disponibilidade | Aceito para o escopo; para HA, 2 tasks em AZs diferentes (ADR 0001) |
| E-mails do Gmail caírem no spam | Aceito para portfólio; adaptador substituível (RNF21) |
| Prazo de 60h | Fatias ordenadas por valor; extras e concorrência têm ordem de corte definida (ver `roadmap.md`) |
