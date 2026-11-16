# Visão do Produto — Barbershop

> Projeto de portfólio para vagas Full Stack (pleno).
> Abordagem: **contract-first** (OpenAPI 3.1) + **SDD com OpenSpec**, com specs criadas incrementalmente, uma fatia por vez.

## 1. Problema

Uma barbearia pequena, com um único barbeiro, gerencia a agenda por mensagens e anotações. Isso gera:

- **Conflitos de horário**: dois clientes marcados no mesmo horário, ou um serviço longo que "invade" o próximo.
- **Tempo perdido do barbeiro** respondendo "tem horário amanhã?" em vez de atender.
- **Faltas e esquecimentos**: o cliente não tem confirmação nem lembrete.
- **Mudanças de agenda difíceis**: folgas, feriados e horários especiais não chegam aos clientes já marcados.

## 2. Objetivo do produto

Permitir que **o cliente agende sozinho, sem criar conta**, um serviço em um horário realmente disponível, e que **o barbeiro controle a própria agenda** (serviços, horários, folgas e cancelamentos) com a garantia de que o sistema nunca aceita um agendamento inválido.

### Objetivos do portfólio

| Objetivo | Como será demonstrado |
|---|---|
| Regras de negócio bem resolvidas | Cálculo de disponibilidade (passo de 15 min, intervalo, pausas, antecedências, ajustes por dia), bloqueio de alteração com conflito, snapshot |
| Testes automatizados | Unitários de domínio (100%), integração com Postgres real, contrato e concorrência |
| Boas práticas de API | Contract-first com OpenAPI 3.1, validação a partir do contrato, erros padronizados, rate limit |
| Infraestrutura na AWS | API em ECS Fargate atrás de um ALB, worker de e-mail em Lambda, PostgreSQL no RDS, ElastiCache, SQS e CloudWatch, executados no emulador oficial da AWS (ver ADR 0001); CI no GitHub Actions |

## 3. Atores

| Ator | Descrição | Pode |
|---|---|---|
| **Cliente** | Pessoa que quer ser atendida. **Não tem conta.** Identifica-se por nome e telefone (obrigatórios) e e-mail (opcional). | Ver serviços, ver horários disponíveis, agendar, cancelar o próprio agendamento (pelo link do e-mail ou pelo site no mesmo navegador) |
| **Barbeiro/Admin** | Dono da barbearia e único barbeiro. Um único usuário, criado via seed. | Fazer login, gerenciar serviços, definir o horário semanal (com pausas), ajustar dias específicos e folgas, configurar regras (intervalo, antecedências, limite por telefone), ver a agenda, cancelar agendamentos, receber aviso de novo agendamento |
| **Sistema (job)** | Processo disparado periodicamente por um agendador (mecanismo a definir em ADR). | Enviar lembretes por e-mail |

Não existe recepcionista.

## 4. Escopo do MVP (resumo)

- Catálogo de serviços com duração e preço fixos. Combos são serviços próprios. O preço é apenas exibido.
- Horário semanal com pausas, ajustes por dia e folgas, gerenciados pelo admin.
- Disponibilidade com início flexível em passos de 15 min e intervalo entre atendimentos configurável.
- Agendamento sem conta; um agendamento tem exatamente um serviço.
- Cancelamento pelo cliente (token de uso único) e pelo barbeiro.
- E-mails: confirmação com link de cancelamento e aviso de cancelamento feito pelo barbeiro.
- Proteções: rate limit por IP e limite de agendamentos ativos por telefone.
- **Extras, se houver tempo**: "meus próximos agendamentos" via localStorage, aviso em tempo real no painel, lembrete por e-mail 2h antes e testes de contrato (Schemathesis).

## 5. Fora de escopo (MVP)

| Item | Observação |
|---|---|
| Histórico de agendamentos do cliente | Implementação futura |
| Conta/login do cliente | Implementação futura |
| Múltiplos barbeiros e separação dono × barbeiro | Implementação futura; hoje dono = barbeiro = admin |
| Multi-tenant (várias barbearias) | O sistema deve ser **preparado** para isso (ver seção 7) |
| Barbeiro agendar em nome do cliente / encaixe | Atendimento sem agendamento acontece fora do sistema |
| Falta do barbeiro como fluxo próprio | O barbeiro usa folga + cancelamento manual |
| Status "concluído" e "não compareceu" | O MVP tem só `AGENDADO` e `CANCELADO` |
| Controle de atraso do cliente | Fica fora do sistema |
| Pagamento online / sinal | O preço é apenas exibido |
| Notificações por WhatsApp/SMS | Futuro (ex.: WhatsApp Cloud API como novo adaptador de notificação) |
| Web Push | Futuro |
| Recuperação de senha do admin | Futuro |
| Escolha de barbeiro pelo cliente | Não se aplica (barbeiro único) |

## 6. Restrições

- **Prazo**: ~60 horas (30 h/semana × 2 semanas).
- **Infraestrutura AWS no emulador oficial**: o projeto serve para praticar AWS e roda no emulador, sem conta real. Serviços: ECS Fargate + ALB (API), Lambda (worker de e-mail), RDS PostgreSQL, ElastiCache, SQS e CloudWatch (ver ADR 0001). E-mail via Gmail API (HTTPS); CI no GitHub Actions. A hospedagem do front está a definir.
- **Idioma**: documentação em português; código, rotas, campos e códigos de erro em inglês.
- **Repositórios separados**: `barbershop-api` (contém o contrato OpenAPI) e o repositório do front (React + Vite, com cliente e admin na mesma aplicação).

## 7. Preparação para multi-tenant — decisão pendente

A decisão será registrada em ADR **antes da Fatia 1**, porque afeta o schema.

- **Opção A — `tenant_id` desde já**, com um tenant fixo criado via seed. A migração futura fica trivial, mas o campo aparece em toda query e todo teste desde o início.
- **Opção B — sem `tenant_id`**, com o acesso a dados isolado em repositórios que recebem um contexto, de modo que o ponto onde o filtro entrará já existe. O modelo fica mais limpo hoje, mas a migração futura exige alterar tabelas e fazer backfill.

Padrões a estudar: tenant context por requisição (`AsyncLocalStorage`), Row Level Security no Postgres e repository pattern.
Referências: Azure Architecture Center ("Architect multitenant solutions"), AWS "SaaS Tenant Isolation Strategies", documentação do PostgreSQL ("Row Security Policies").
