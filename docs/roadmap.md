# Roadmap — Fatias Verticais

> Cada fatia entrega valor **de ponta a ponta** (contrato → API → banco → front → deploy) e vira **um change no OpenSpec**, criado só quando a fatia começar.
> Orçamento: ~60h. As estimativas incluem testes.

## Fluxo de cada fatia

1. **Proposta OpenSpec** (`openspec/changes/<nome>`): objetivo, RF/RN cobertos, critérios de aceite.
2. **Contrato primeiro**: alterar `openapi.yaml`, passar no Spectral e revisar.
3. **Front contra o mock** (Prism) com client gerado, em paralelo à API.
4. **Testes de domínio** → implementação → testes de integração.
5. **Deploy** na `main` e verificação no ambiente publicado.
6. **Arquivar o change** no OpenSpec, atualizando as specs consolidadas.

---

## Núcleo

### Fatia 0 — Esqueleto andando · ~6h
**Entrega:** os dois repositórios publicados e conversando.
- Repositórios, TypeScript strict, lint e CI (Spectral, lint, tipos, testes, cobertura).
- `openapi.yaml` inicial com `GET /v1/health`; validação pelo contrato no Fastify.
- Conexão com o Supabase, ferramenta de migrations e Testcontainers funcionando.
- Front com Vite publicado na Vercel, chamando `/health` via client gerado.
- **Pré-requisito:** ADR de multi-tenant (Q5) e ADR de stack.
- **Cobre:** RNF01–RNF04, RNF12, RNF17, RNF26–RNF29.

### Fatia 1 — Cliente agenda um horário · ~8h  ⭐ menor fatia de valor
**Entrega:** o cliente escolhe um serviço, vê horários livres e agenda. Serviços, horário semanal e configurações vêm do **seed** (sem tela de admin ainda).
- Listar serviços, calcular disponibilidade e criar agendamento com snapshot.
- Validações: passo de 15 min, cabe no período, sem sobreposição (em nível de aplicação), antecedências, telefone.
- Erros em Problem Details.
- **Decidir antes:** Q1, Q2.
- **Cobre:** RF01–RF03 · RN01–RN09, RN11, RN20, RN24.

### Fatia 2 — Barbeiro vê e cancela a agenda · ~6h
**Entrega:** o admin faz login, vê os agendamentos e cancela.
- Login (seed do admin, hash de senha, JWT); agenda por período; cancelamento com registro de quem e quando.
- **Cobre:** RF07, RF12, RF13 · RN18, RN19, RN25 · RNF13.

### Fatia 3 — Catálogo gerenciado pelo admin · ~4h
**Entrega:** o admin cria, edita e desativa serviços; o catálogo público reflete as mudanças.
- **Decidir antes:** Q3, Q4.
- **Cobre:** RF08 · RN03, RN15, RN16.

### Fatia 4 — Horário semanal e configurações · ~9h
**Entrega:** o admin define o horário semanal com pausas e as configurações; alterações em conflito são bloqueadas e os conflitos, listados.
- **Cobre:** RF09, RF11 · RN11, RN13, RN14.

### Fatia 5 — Ajustes por data e folgas · ~4h
**Entrega:** o admin ajusta dias específicos, marca folgas e remove ajustes, sempre respeitando o bloqueio por conflito.
- **Cobre:** RF10 · RN12, RN13.

### Fatia 6 — Cancelamento pelo cliente e e-mails · ~7h
**Entrega:** o cliente recebe o e-mail de confirmação com o link, cancela pelo link e é avisado quando o barbeiro cancela.
- Adaptador `EmailSender` com Gmail API (setup OAuth ~1h); token de uso único, armazenado como hash, que expira no início do agendamento; tela de cancelamento.
- **Cobre:** RF04, RF05, RF15, RF16 · RN17, RN19, RN21 · RNF14, RNF21.

### Fatia 7 — Proteções contra abuso · ~2h
**Entrega:** rate limit por IP e limite configurável de agendamentos ativos por telefone.
- **Cobre:** RN10 · RNF15.

**Subtotal do núcleo: ~46h.**

---

## Fatia de robustez — primeiro corte se o prazo apertar

### Fatia 8 — Dupla reserva impossível sob concorrência · ~3h
**Entrega:** exclusion constraint no Postgres (`tstzrange` da ocupação, somente agendamentos ativos), tradução da violação para `SLOT_UNAVAILABLE` e teste com N requisições simultâneas.
- **Cobre:** RN07 (sob concorrência) · RNF11.

---

## Extras do MVP (em ordem de prioridade)

| Fatia | Entrega | ~h | Cobre |
|---|---|---|---|
| 9 | "Meus próximos agendamentos" via localStorage, com botão de cancelar | 2 | RF06 |
| 10 | Aviso de novo agendamento em tempo real no painel (SSE ou polling) | 3 | RF14 · RN23 |
| 11 | Lembrete por e-mail 2h antes: endpoint interno idempotente + workflow agendado no GitHub Actions | 4 | RF17 · RN22 · RNF16, RNF31 |
| 12 | Testes de contrato com Schemathesis no CI | 3 | RNF07 |

**Total com todas as fatias: ~61h.** O plano não tem folga: os extras só entram se o núcleo fechar no prazo.

---

## Depois do MVP (futuro)

1. Histórico do cliente e contas de cliente (RF-F01, RF-F02)
2. Múltiplos barbeiros e separação de papéis (RF-F03)
3. Multi-tenant (RF-F04)
4. Status "concluído" e "não compareceu" e encaixe (RF-F05, RF-F06)
5. WhatsApp Cloud API e Web Push como novos adaptadores de notificação (RF-F07, RF-F08)
6. Recuperação de senha (RF-F09)
7. Pagamento/sinal (RF-F10)
