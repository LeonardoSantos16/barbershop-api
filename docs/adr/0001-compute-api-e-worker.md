# ADR 0001 — Compute da API e do worker de e-mail

- **Status:** Aceito
- **Data:** 2026-11-16
- **Substitui:** a parte de compute do RNF26 (API no Render free)

## Contexto

A infraestrutura passa a ser AWS, executada no emulador oficial da AWS, para praticar os serviços. O plano anterior (Render, Supabase e Vercel com custo zero) foi abandonado.

Há duas cargas de trabalho:

1. **API HTTP** (Fastify, Node.js): endpoints públicos do cliente e endpoints autenticados do admin. Acessa o RDS PostgreSQL e o ElastiCache (cache de "meus agendamentos" e contador de rate limit por IP).
2. **Worker de e-mail**: consome a fila SQS e envia os e-mails transacionais (RN21) pela Gmail API.

Restrições relevantes:

- Volume baixo: uma barbearia com um barbeiro.
- O cold start já era aceito no plano anterior (RNF30).
- O RF14 (aviso em tempo real) é Extra e **fica fora deste ADR**.

## Opções consideradas

### API

| Critério | Lambda + API Gateway | ECS Fargate + ALB |
|---|---|---|
| Conexões com o Postgres | Uma por ambiente de execução. Ambientes congelados mantêm conexões abertas. Um pico anormal (~75 ambientes) esgota o banco, o que exige `reserved concurrency` e pool mínimo, ou RDS Proxy (+US$ 22/mês) | Fixas e previsíveis: tasks × tamanho do pool (1 × 10) |
| Estado em processo (pool, caches locais, conexões longas) | Efêmero, por ambiente | Processo contínuo |
| Cold start | Existe, mas é aceitável (RNF30) | Não existe com a task em execução |
| Custo mensal estimado (us-east-1) | ~US$ 27 no total, compute ~US$ 0 | ~US$ 64 no total: Fargate ~US$ 9, ALB ~US$ 17, IPv4 públicos ~US$ 11 |
| Empacotamento | Adaptador Fastify→Lambda e handler específico | Imagem Docker, igual à execução local |
| Objetivo de estudo | Serverless | ECS, ALB, VPC, task definitions e deploy de containers |

### Worker

| Critério | Lambda (event source mapping do SQS) | Task Fargate fazendo polling |
|---|---|---|
| Cold start | Irrelevante: ninguém espera a resposta | — |
| Polling, lotes e retentativas | Gerenciados pela AWS | Implementados no código |
| Custo em repouso | ~US$ 0 | Task sempre ligada |

## Decisão

1. **API em ECS Fargate atrás de um Application Load Balancer.**
   - 1 task, 0,25 vCPU / 0,5 GB, arquitetura ARM64 (Graviton).
   - Pool do `pg` com `max: 10`.
   - Health check do ALB e do ECS apontando para `/health` (RNF28).
   - Deploy rolling com `minimumHealthyPercent: 100` e `maximumPercent: 200`: a nova task sobe e fica saudável antes de a antiga sair.
2. **Worker em AWS Lambda, acionado pelo SQS e fora da VPC.**
   - A mensagem carrega tudo o que o e-mail precisa (destinatário, dados do agendamento, link de cancelamento). Por isso o worker não acessa o RDS.
   - Fora da VPC, o worker alcança a Gmail API e o Secrets Manager sem NAT Gateway.
3. **Rede.**
   - RDS e ElastiCache em subnets privadas.
   - Tasks Fargate em subnets públicas com IP público, para alcançar ECR, SQS, e CloudWatch Logs **sem NAT Gateway**.
   - O security group da task aceita entrada **somente** do security group do ALB.
   - Os security groups do RDS e do ElastiCache aceitam entrada **somente** do security group da task.

## Justificativa

- **Previsibilidade das conexões** com o banco que garante a RN07, sem RDS Proxy nem ajuste fino de concorrência.
- **Objetivo de estudo:** containers, ECS e ALB são o modelo mais comum em APIs de produção. A mesma imagem roda localmente e na AWS.
- O custo extra (~US$ 37/mês a mais que a opção Lambda, vindo quase todo do ALB e dos IPv4 públicos) é **aceito conscientemente** como custo de aprendizado. No emulador, nada disso é cobrado.
- O worker é assíncrono, então o cold start não afeta ninguém. A integração nativa SQS→Lambda elimina código de polling e retentativa. Tirá-lo da VPC elimina a única razão para ter um NAT Gateway.

## Consequências

**Positivas**

- No máximo 10 conexões da API no banco, previsível mesmo sob ataque. O rate limit protege a aplicação, e o banco continua protegido por construção.
- Nenhum NAT Gateway (economia de ~US$ 36/mês).
- Retentativas, DLQ e paralelismo do worker são configuração, não código.

**Negativas / riscos**
- **Custo fixo** de ~US$ 64/mês mesmo sem tráfego.
- **Sem alta disponibilidade:** uma task só. Se ela cair, a API fica indisponível até o ECS subir outra. Aceito para o escopo. Para HA, usar 2 tasks em AZs diferentes.
- **Task em subnet pública:** a proteção depende inteiramente dos security groups. Revisar as regras de entrada é obrigatório.
- **E-mail duplicado:** o SQS entrega pelo menos uma vez e o worker não acessa o banco para deduplicar. Uma confirmação duplicada é rara e tolerável. A idempotência dos lembretes (RN22) fica do lado do produtor, que roda dentro da VPC e marca o envio no banco antes de enfileirar.
- **Dados pessoais na mensagem:** o e-mail e o nome do cliente ficam na fila. Usar criptografia do SQS (SSE) e um período de retenção curto.
- **Consistência banco × fila:** a publicação no SQS acontece depois do commit no Postgres. Se a publicação falhar, o e-mail se perde. Avaliar o padrão *transactional outbox* em ADR próprio.

## Pendências

- Verificar o suporte do emulador a ECS/Fargate, ALB e ao event source mapping SQS→Lambda.
- Disparo do job de lembretes (RF17/RN22): EventBridge Scheduler no lugar do cron do GitHub Actions (RNF31), em ADR próprio.
- RF14 (tempo real): decidir na fatia correspondente. Com Fargate, SSE é viável; com mais de uma task, exige pub/sub.