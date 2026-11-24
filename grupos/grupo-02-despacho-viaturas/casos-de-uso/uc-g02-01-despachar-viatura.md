# UC-G02-01 — Despachar Viatura

## Identificação

| Campo | Valor |
| --- | --- |
| **ID** | UC-G02-01 |
| **Módulo** | G02 — Despacho e Frota de Viaturas |
| **Ator primário** | MedicoRegulador |
| **Atores secundários** | EquipeIntervencao, Módulo de Telemedicina (G3), Módulo de Auditoria (G6) |
| **Objetivo** | Alocar uma viatura compatível (USB ou USA) e sua respectiva equipe de intervenção a um chamado previamente classificado pelo Protocolo de Manchester. |
| **Requisitos rastreados** | RF-G02-01, RF-G02-02, RF-G02-03, RF-G02-04, RF-G02-07, RF-G02-08 |
| **Prioridade** | essencial |

## Pré-condições

- O evento `chamado_classificado` (Contrato C1) foi recepcionado do módulo G1 contendo `chamado_id`, `prioridade` válida, `queixa_principal`, `localizacao` e `triagem_em`.
- O chamado encontra-se na fila de despacho aguardando alocação de recurso móvel.

## Pós-condições

- **Sucesso:** Um registro `Despacho` é criado com identificador único (`despacho_id`), a viatura selecionada transita para o estado `Deslocando para Sinistro`, o chamado muda para `Viatura Despachada`, o contrato `despacho_ativo` (C2) é publicado para o módulo G3 e o evento sensível `VIATURA_DESPACHADA` (C5) é enviado para o módulo G6.
- **Falha:** Nenhuma viatura é vinculada ao chamado, o estado original das viaturas permanece inalterado, o chamado permanece na fila de despacho (ou retorna para triagem em caso de dado inválido) e o erro ou exceção é registrado.

## Fluxo principal

1. O MedicoRegulador seleciona um chamado classificado na fila de despacho ordenada por prioridade clínica.
2. O sistema executa o caso de uso incluído `UC-G02-05 — Verificar Disponibilidade de Equipe` para filtrar viaturas no estado `Disponível` com tripulação completa.
3. O sistema recomenda o tipo de viatura (`USA` para prioridades `VERMELHO`/`LARANJA`; `USB` para `AMARELO`/`VERDE`/`AZUL`) e ordena as opções disponíveis pela menor distância até a `localizacao` do chamado.
4. O MedicoRegulador seleciona a viatura recomendada e registra a justificativa clínica/operacional da escolha.
5. O sistema valida o bloqueio exclusivo da viatura (impedindo alocação concorrente), altera o estado da viatura para `Deslocando para Sinistro` e vincula a `Equipe` e o `Chamado` ao novo `Despacho`.
6. O sistema publica o evento `despacho_ativo` (Contrato C2) para o módulo G3 e o evento sensível `VIATURA_DESPACHADA` (Contrato C5) para o módulo G6.
7. O sistema notifica a EquipeIntervencao embarcada sobre os dados da ocorrência e exibe a confirmação do despacho ao MedicoRegulador.

## Fluxos alternativos

### FA-01 — Seleção manual de tipo de viatura divergente da recomendação

- **Condição:** No passo 4, o MedicoRegulador opta por despachar uma viatura de complexidade diferente da sugerida automaticamente (ex.: enviar `USB` em chamado `LARANJA` por indisponibilidade imediata de `USA` próxima, ou enviar `USA` em chamado `AMARELO` por agravamento relatado).
- **Passos:**
  1. O MedicoRegulador seleciona manualmente outra viatura `Disponível` na lista.
  2. O sistema exige o preenchimento obrigatório de justificativa clínica para a alteração de modalidade.
  3. O MedicoRegulador informa a justificativa e confirma a operação.
- **Reconvergência:** retorna ao passo 5 do fluxo principal.

## Fluxos de exceção

### FE-01 — Indisponibilidade de viatura compatível na frota

- **Condição:** No passo 2 ou 3, não existem viaturas compatíveis no estado `Disponível` com equipe completa.
- **Passos:**
  1. O sistema exibe alerta crítico de indisponibilidade de frota para o MedicoRegulador.
  2. O sistema devolve o chamado à fila prioritária de espera e oferece a opção de redirecionar o caso para teleorientação ampliada no módulo G3.
  3. O MedicoRegulador registra a ciência da indisponibilidade e a conduta adotada.
- **Estado final:** Chamado permanece aguardando recurso na fila de despacho sem viatura alocada; registro de alerta gravado no histórico operacional.

### FE-02 — Conflito de alocação concorrente da mesma viatura

- **Condição:** No passo 5, outro regulador aloca a mesma viatura milissegundos antes da confirmação.
- **Passos:**
  1. O sistema detecta que o estado da viatura já foi alterado para `Deslocando para Sinistro`.
  2. O sistema aborta a transação, notifica o MedicoRegulador sobre a alocação concorrente e atualiza a lista de viaturas disponíveis.
- **Estado final:** Nenhum despacho duplicado é gerado; o chamado permanece na fila para nova seleção.

### FE-03 — Violação do contrato de entrada (chamado sem prioridade)

- **Condição:** Antes do passo 1, o evento `chamado_classificado` chega sem o campo obrigatório `prioridade` preenchido.
- **Passos:**
  1. O sistema bloqueia o chamado para despacho operacional.
  2. O sistema emite alerta ao MedicoRegulador informando inconsistência na triagem.
- **Estado final:** Chamado mantido no estado `Em Triagem` até regularização pelo módulo G1.

## Regras de negócio aplicadas

| Regra | Descrição |
| --- | --- |
| RN-03 | Uma viatura de Suporte Avançado (`USA`) só pode ser despachada se possuir equipe avançada completa vinculada (Médico, Enfermeiro e Condutor Socorrista). |
| RN-04 | É proibida a dupla alocação simultânea da mesma viatura em dois despachos ativos distintos. |
| Falha de Contrato C1 | Sem o campo `prioridade` definido no contrato `chamado_classificado`, o módulo G2 não realiza o despacho e alerta o regulador. |

## Relacionamentos

- `<<include>>` `UC-G02-05` (Verificar Disponibilidade de Equipe) — obrigatório porque nenhuma viatura pode ser despachada sem a validação prévia de que sua equipe mínima está ativa e apta para intervenção (RN-03).
- `<<extend>>` `UC-G02-02` (Acionar Suporte Avançado) — opcional porque ocorre somente quando há necessidade de apoio adicional de uma `USA` durante ou após o despacho inicial devido a agravamento clínico em campo.

## Dados manipulados

| Entidade | Dado | Origem |
| --- | --- | --- |
| Chamado | `chamado_id`, `prioridade`, `queixa_principal`, `localizacao`, `triagem_em` | G1 (Contrato C1) |
| Viatura | `viatura_id`, `tipo` (`USB`/`USA`), `estado`, `localizacao_atual` | G2 |
| Equipe | `equipe_id`, `membros`, `status_disponibilidade` | G2 |
| Despacho | `despacho_id`, `justificativa`, `eta_local`, `timestamp_despacho` | G2 |

## Critérios de aceite

- [ ] O sistema impede o despacho de chamados sem `prioridade` classificada.
- [ ] O sistema recomenda `USA` para prioridades `VERMELHO` e `LARANJA`, e `USB` para as demais, ordenando por proximidade.
- [ ] O sistema bloqueia qualquer tentativa de despachar uma viatura que não esteja no estado `Disponível` (ex.: `Em Descontaminação/Reposição` ou já despachada).
- [ ] Ao confirmar o despacho, o evento `despacho_ativo` (C2) e o evento sensível `VIATURA_DESPACHADA` (C5) são emitidos corretamente.

## Rastreabilidade

| RF | Classe | Sequência | Teste |
| --- | --- | --- | --- |
| RF-G02-01 | `Despacho`, `Chamado` | DS-G02-01 | CT-G02-01 |
| RF-G02-02 | `Viatura`, `ViaturaUSA`, `ViaturaUSB`, `Base` | DS-G02-01 | CT-G02-01 |
| RF-G02-03 | `Despacho`, `Viatura`, `Equipe` | DS-G02-01 | CT-G02-02 |
| RF-G02-07 | `Despacho`, `Viatura` | DS-G02-02 | CT-G02-03 |