# UC-G02-03 — Encerrar Atendimento

## Identificação

| Campo | Valor |
| --- | --- |
| **ID** | UC-G02-03 |
| **Módulo** | G02 — Despacho e Frota de Viaturas |
| **Ator primário** | EquipeIntervencao |
| **Atores secundários** | MedicoRegulador, Módulo de Auditoria (G6) |
| **Objetivo** | Finalizar o despacho operacional após a conclusão do atendimento ou entrega do paciente na unidade de destino, encaminhando a viatura para higienização/reposição antes de liberá-la para novas ocorrências. |
| **Requisitos rastreados** | RF-G02-05, RF-G02-08 |
| **Prioridade** | essencial |

## Pré-condições

- Existe um `Despacho` ativo vinculado a uma `Viatura` que se encontra nos estados `No Local` ou `Em Trânsito p/ Hospital`.
- O atendimento pré-hospitalar ao paciente foi concluído (por entrega em unidade hospitalar, liberação no local ou óbito constatado).

## Pós-condições

- **Sucesso:** O `Despacho` é marcado como `Encerrado` com registro do horário de conclusão, a `Viatura` transita obrigatoriamente para o estado `Em Descontaminação/Reposição` (permanecendo bloqueada para novos despachos até a liberação) e o evento `ATENDIMENTO_ENCERRADO` é publicado para o módulo G6.
- **Falha:** O `Despacho` permanece em aberto e a `Viatura` mantém seu estado operacional anterior.

## Fluxo principal

1. A EquipeIntervencao aciona a opção de encerrar atendimento no terminal móvel informando o desfecho operacional (entrega hospitalar ou liberação no local).
2. O sistema valida se o `Despacho` está ativo e se a `Viatura` encontra-se em uma etapa válida para encerramento (`No Local` ou `Em Trânsito p/ Hospital`).
3. O sistema altera o status do `Despacho` para `Encerrado` registrando o timestamp de conclusão.
4. O sistema transita o estado da `Viatura` para `Em Descontaminação/Reposição`, mantendo-a indisponível para novas alocações até a conclusão da higienização.
5. O sistema publica o evento `ATENDIMENTO_ENCERRADO` para o módulo G6 (Auditoria/Compliance) e atualiza o painel de frota do MedicoRegulador.

## Fluxos alternativos

### FA-01 — Encerramento sem transporte hospitalar (alta/recusa no local)

- **Condição:** No passo 1, o paciente é atendido no local e não necessita de remoção para unidade de saúde (ou recusa transporte).
- **Passos:**
  1. A EquipeIntervencao seleciona o motivo `Atendimento concluído no local` a partir do estado `No Local`.
  2. O sistema registra o desfecho sem exigir unidade hospitalar de destino.
- **Reconvergência:** retorna ao passo 3 do fluxo principal.

## Fluxos de exceção

### FE-01 — Tentativa de encerramento em estado operacional inválido

- **Condição:** No passo 2, a `Viatura` ainda está no estado `Deslocando para Sinistro` (não chegou ao local).
- **Passos:**
  1. O sistema bloqueia o encerramento direto e informa que a chegada ao local ainda não foi registrada.
  2. O sistema orienta a equipe a registrar a chegada ou solicitar cancelamento justificado pelo MedicoRegulador.
- **Estado final:** Despacho permanece ativo e `Viatura` segue em `Deslocando para Sinistro`.

## Regras de negócio aplicadas

| Regra | Descrição |
| --- | --- |
| Ciclo de Vida da Viatura | Após encerrar um atendimento, a viatura não retorna diretamente ao estado `Disponível`; ela deve passar por `Em Descontaminação/Reposição` para evitar alocação antes da higienização. |
| Auditoria G6 | Todo encerramento de ocorrência deve publicar o evento `ATENDIMENTO_ENCERRADO` para rastreabilidade. |

## Relacionamentos

- Não possui relacionamentos obrigatórios de `<<include>>` ou `<<extend>>`.

## Dados manipulados

| Entidade | Dado | Origem |
| --- | --- | --- |
| Despacho | `despacho_id`, `status`, `desfecho`, `encerrado_em` | G2 |
| Viatura | `viatura_id`, `estado` | G2 |

## Critérios de aceite

- [ ] Ao encerrar o atendimento, a viatura entra automaticamente em `Em Descontaminação/Reposição` e não aparece na lista de viaturas disponíveis para despacho.
- [ ] O evento `ATENDIMENTO_ENCERRADO` é publicado corretamente para o módulo G6.

## Rastreabilidade

| RF | Classe | Sequência | Teste |
| --- | --- | --- | --- |
| RF-G02-05 | `Viatura`, `Despacho` | DS-G02-03 | CT-G02-05 |
| RF-G02-08 | `Despacho` | DS-G02-03 | CT-G02-05 |