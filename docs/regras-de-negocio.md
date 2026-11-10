# Regras de Negócio

> Todas as regras de data e hora são avaliadas no fuso **America/Sao_Paulo** (RN24).
> Exemplos usam: intervalo = 10 min, antecedência mínima = 60 min, antecedência máxima = 7 dias, salvo indicação em contrário.

## Cliente e agendamento

### RN01 — Identificação do cliente
O cliente não tem conta. Para agendar, informa **nome** (obrigatório), **telefone** (obrigatório, celular brasileiro: DDD válido + 9 dígitos começando com 9) e **e-mail** (opcional; se informado, deve ser válido).
- ✅ `Ana`, `11987654321`, sem e-mail
- ❌ `Ana`, `1187654321` (8 dígitos) → telefone inválido
- ❌ sem nome → rejeitado

### RN02 — Um serviço por agendamento
Cada agendamento tem **exatamente um** serviço ativo do catálogo. Combinações existem como serviços próprios (ex.: "Corte + Sombrancelha + Barba").
- ❌ Agendar "Corte" e "Barba" no mesmo agendamento → rejeitado; o cliente escolhe o combo.

### RN03 — Snapshot do serviço e das configurações
Ao criar o agendamento, o sistema guarda uma cópia de **nome, duração e preço do serviço** e do **intervalo vigente**. Alterações posteriores não afetam agendamentos existentes.
- Exemplo: "Luzes" de 90 min é agendado; depois o admin muda para 120 min. O agendamento existente continua com 90 min; os novos usam 120.

### RN04 — Ocupação de um agendamento
O **atendimento** vai de `início` até `início + duração`. A **ocupação** vai de `início` até `início + duração + intervalo` (intervalo em snapshot).
- Corte de 30 min às 10h00, intervalo de 10 min → atendimento 10h00–10h30, ocupação 10h00–10h40.

### RN05 — Horários de início em passos de 15 min
Horários de início só podem ser `hh:00`, `hh:15`, `hh:30` ou `hh:45`.
- ✅ 10h15 · ❌ 10h10

### RN06 — O atendimento deve caber em um período de trabalho
O **atendimento** (sem o intervalo) deve estar inteiramente contido em **um único** período de trabalho do dia. Ele pode terminar exatamente no fim do período (fim do expediente ou início da pausa). O intervalo **não** precisa caber depois do último atendimento de um período.
- Expediente 09h–12h e 13h–19h, corte de 30 min:
  - ✅ 18h30 (termina às 19h00)
  - ✅ 11h30 (termina às 12h00, quando a pausa começa)
  - ❌ 11h45 (invadiria a pausa)
  - ✅ 13h00 (começa no fim da pausa)

### RN07 — Sem sobreposição de ocupações
Duas ocupações de agendamentos **ativos** não podem se sobrepor (intervalos semiabertos `[início, fim)`).
- Existe um corte às 10h00 (ocupação até 10h40). Um corte às 10h30 é ❌; às 10h45 é ✅ (10h40 não é múltiplo de 15).
- Esta regra deve valer **também sob requisições simultâneas** (ver Fatia 8 do roadmap).

### RN08 — Antecedência mínima
O início do agendamento deve ser ≥ `agora + antecedência mínima` (configurável pelo admin, em minutos).
- Agora 14h05, mínimo de 60 min → o primeiro horário possível é 15h15 (15h05 arredondado para o próximo passo de 15 min).

### RN09 — Antecedência máxima
O início do agendamento deve ser ≤ `hoje + antecedência máxima` (configurável pelo admin, em dias; padrão 7).
- Hoje é 10/11 e o máximo é 7 dias → é possível agendar até o fim do dia 17/11. *(A semântica "até o fim do dia" está pendente de confirmação; ver Questões em aberto.)*

### RN10 — Limite de agendamentos por telefone
Um telefone pode ter no máximo **N agendamentos ativos e futuros** (N configurável pelo admin). Cancelados e passados não contam.
- N = 1: o cliente com um corte marcado para amanhã não consegue marcar outro; se cancelar, consegue.

## Agenda do barbeiro

### RN11 — Horário semanal recorrente
Para cada dia da semana, o admin define **zero ou mais períodos de trabalho** (início < fim, sem sobreposição entre eles). Um dia sem períodos é um dia fechado. Pausas são o espaço entre os períodos.
- Seg–Sex: 09h–12h e 13h–19h · Sáb: 09h–14h · Dom: fechado.

### RN12 — Ajuste por data
O admin pode definir, para uma data específica, períodos que **substituem integralmente** o horário semanal daquela data. Um ajuste sem períodos é uma **folga** (feriado, dia livre), e não exige motivo.
- O ajuste sempre vence o horário semanal.
- Alterar o horário semanal **não** afeta datas que já têm ajuste.
- O admin pode **remover** o ajuste; a data volta a seguir o horário semanal.
- Exemplo: o horário semanal de quarta é 09h–19h; o ajuste de 12/11 é 11h–14h → em 12/11 só existem horários entre 11h e 14h.

### RN13 — Alteração de horário com agendamentos em conflito é bloqueada
Qualquer alteração de horário (semanal, ajuste, folga ou remoção de ajuste) é **rejeitada** se algum agendamento ativo e futuro deixar de satisfazer a RN06 com os novos períodos. A resposta lista os agendamentos em conflito. O barbeiro precisa cancelá-los antes (RN18) e só então repetir a alteração.
- Existe um corte em 12/11 às 16h; o barbeiro tenta definir folga em 12/11 → rejeitado, com o conflito listado.

### RN14 — Configurações afetam apenas novos agendamentos
Mudanças em intervalo, antecedência mínima, antecedência máxima e limite por telefone valem só para agendamentos criados depois da mudança.

## Catálogo

### RN15 — Serviço
Um serviço tem nome (obrigatório), duração em minutos (> 0) e preço (≥ 0, apenas para exibição). O admin pode criar, editar e desativar serviços.
- Serviços iniciais (seed): Corte; Corte + Sombrancelha; Corte + Sombrancelha + Barba; Barba; Luzes; Bigode + Cavanhaque; Bigode + Cavanhaque + Cabelo; Bigode + Cabelo.

### RN16 — Desativação de serviço
Um serviço desativado **sai do catálogo público** e não pode ser usado em novos agendamentos. Agendamentos existentes com ele continuam válidos (RN03).

## Cancelamento

### RN17 — Cancelamento pelo cliente
O cliente cancela usando o **token de cancelamento** do agendamento, recebido pelo link do e-mail ou guardado no localStorage do navegador onde agendou.
- O token é aleatório, de **uso único**, e **expira no horário de início** do agendamento.
- O cancelamento só é aceito **até o horário de início**.
- Depois do uso, o token fica inválido no e-mail e no site (o mesmo token é entregue pelos dois canais).
- O horário é liberado imediatamente.
- Exemplo: corte às 15h; cancelar às 14h59 ✅; às 15h01 ❌ (token expirado).

### RN18 — Cancelamento pelo barbeiro
O barbeiro pode cancelar qualquer agendamento ativo e futuro pelo painel, **sem motivo**.

### RN19 — Registro do cancelamento
Todo cancelamento registra **quem cancelou** (`CLIENT` ou `BARBER`) e **quando**. Não há motivo.

### RN20 — Status
O agendamento tem apenas os status `SCHEDULED` e `CANCELLED`. Um agendamento cancelado não pode ser reativado.

## Notificações

### RN21 — E-mails transacionais
Enviados somente se o cliente informou e-mail:
- **Confirmação** ao agendar, com data, hora, serviço, preço e **link de cancelamento**.
- **Aviso de cancelamento** quando o barbeiro cancela.
- O cancelamento feito pelo próprio cliente **não** gera e-mail.
- Cliente sem e-mail que precisa cancelar sem acesso ao navegador original deve contatar o barbeiro.

### RN22 — Lembrete 2h antes *(extra do MVP)*
O lembrete é enviado por e-mail aproximadamente **2h antes** do início, somente se:
- o agendamento está ativo;
- o cliente informou e-mail;
- o agendamento foi criado com **2h ou mais** de antecedência;
- o lembrete ainda não foi enviado (envio idempotente).

O envio é disparado por um job periódico; um atraso de alguns minutos é aceitável.

### RN23 — Aviso de novo agendamento ao barbeiro *(extra do MVP)*
Quando surge um novo agendamento, o painel aberto do barbeiro mostra um aviso sem precisar recarregar a página.

## Gerais

### RN24 — Fuso horário único
Todas as regras de data e hora usam **America/Sao_Paulo**. As datas são armazenadas em UTC.

### RN25 — Administrador único
Existe um único admin (o barbeiro), criado via seed, que faz login com e-mail e senha. Todas as operações administrativas exigem autenticação.

---

## Questões em aberto (decidir antes da fatia correspondente)

| # | Questão | Fatia |
|---|---|---|
| Q1 | Antecedência máxima de 7 dias: contar "até o fim do 7º dia" ou "até agora + 7×24h"? | 1 |
| Q2 | Valores padrão de intervalo, antecedência mínima e limite por telefone? | 1 / 4 |
| Q3 | A duração do serviço precisa ser múltipla de 15 min? (Hoje não há restrição.) | 3 |
| Q4 | Um serviço desativado pode ser reativado? | 3 |
| Q5 | Multi-tenant: opção A (`tenant_id` já) ou B (isolar pelos repositórios)? | antes da 1 |
