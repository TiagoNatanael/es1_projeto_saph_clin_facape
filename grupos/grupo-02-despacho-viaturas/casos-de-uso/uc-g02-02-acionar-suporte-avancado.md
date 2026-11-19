# UC-G02-02 — Acionar Suporte Avançado

## Identificação

| Campo | Valor |
| --- | --- |
| **ID** | UC-G02-02 |
| **Módulo** | G02 — Despacho e Frota de Viaturas |
| **Ator primário** | MedicoRegulador |
| **Atores secundários** | EquipeIntervencao, Módulo de Telemedicina (G3), Módulo de Auditoria (G6) |
| **Objetivo** | Alocar uma Unidade de Suporte Avançado (USA) em apoio a um chamado já em atendimento ou despachado quando houver agravamento clínico ou necessidade de recurso avançado. |
| **Requisitos rastreados** | RF-G02-02, RF-G02-03, RF-G02-05, RF-G02-06, RF-G02-07, RF-G02-08 |
| **Prioridade** | essencial |

## Pré-condições

- Existe um chamado previamente aberto e vinculado a um despacho ou atendimento em curso (`chamado_id` ativo)[cite: 13].
- Há demanda clínica de suporte avançado (ex.: agravamento do quadro, parada cardiorrespiratória no local ou necessidade de intervenção médica avançada)[cite: 20].

## Pós-condições

- **Sucesso:** Uma nova alocação de `ViaturaUSA` com equipe completa é registrada vinculada ao mesmo `chamado_id`, a viatura de suporte avançado transita para `Deslocando para Sinistro`, o contrato `despacho_ativo` (C2) é atualizado/publicado para o módulo G3 e o evento sensível `SUPORTE_AVANCADO_ACIONADO` (C5) é emitido para o módulo G6[cite: 13, 15, 16, 20, 21].
- **Falha:** Nenhuma `ViaturaUSA` é vinculada ao chamado, o atendimento atual permanece sem reforço avançado e a indisponibilidade ou falha é registrada para decisão do regulador.

## Fluxo principal

1. A EquipeIntervencao em campo ou o MedicoRegulador identifica agravamento clínico que exige Unidade de Suporte Avançado (USA) e solicita o apoio avançado vinculando o `chamado_id`[cite: 13, 20].
2. O sistema executa o caso de uso incluído `UC-G02-05 — Verificar Disponibilidade de Equipe` filtrando exclusivamente viaturas do tipo `USA` no estado `Disponível` com equipe avançada completa (RN-03)[cite: 20, 21, 22].
3. O sistema lista as `ViaturaUSA` disponíveis ordenadas pela menor distância/ETA até a `localizacao` do chamado[cite: 13, 18, 22].
4. O MedicoRegulador seleciona a `ViaturaUSA` e registra a justificativa do acionamento de suporte avançado[cite: 20, 22].
5. O sistema bloqueia a viatura contra alocação concorrente, altera o estado da `ViaturaUSA` para `Deslocando para Sinistro` e associa o novo despacho ao `chamado_id` existente[cite: 13, 21, 22].
6. O sistema publica o evento `despacho_ativo` (Contrato C2) para o módulo G3 e o evento sensível `SUPORTE_AVANCADO_ACIONADO` (Contrato C5) para o módulo G6[cite: 13, 15, 16].
7. O sistema notifica a equipe da `ViaturaUSA` e confirma o acionamento na tela do MedicoRegulador[cite: 20, 22].

## Fluxos alternativos

### FA-01 — Solicitação iniciada diretamente pela EquipeIntervencao via terminal de campo

- **Condição:** No passo 1, a solicitação de apoio avançado é disparada pela EquipeIntervencao (ex.: tripulação de uma `USB` no local) antes do contato verbal com a regulação[cite: 20, 22].
- **Passos:**
  1. O sistema registra a solicitação de apoio avançado no `chamado_id` e gera alerta prioritário na fila do MedicoRegulador[cite: 13, 20].
  2. O MedicoRegulador revisa os dados clínicos reportados e valida a necessidade de `USA`[cite: 20, 22].
- **Reconvergência:** retorna ao passo 2 do fluxo principal.

## Fluxos de exceção

### FE-01 — Nenhuma ViaturaUSA disponível na região

- **Condição:** No passo 2 ou 3, não existe nenhuma `ViaturaUSA` no estado `Disponível` com equipe completa[cite: 20, 21, 22].
- **Passos:**
  1. O sistema exibe alerta de indisponibilidade de `USA` ao MedicoRegulador[cite: 20, 22].
  2. O sistema apresenta alternativas: manter atendimento com a equipe atual sob teleorientação médica intensiva (G3) ou solicitar apoio de base mais distante[cite: 19, 20].
  3. O MedicoRegulador registra a decisão tomada e a justificativa obrigatória[cite: 15, 20].
- **Estado final:** Nenhuma `ViaturaUSA` é alocada; o chamado mantém o atendimento ativo anterior e o registro de indisponibilidade fica salvo para auditoria[cite: 15, 21].

### FE-02 — Conflito de alocação concorrente da mesma ViaturaUSA

- **Condição:** No passo 5, a `ViaturaUSA` escolhida já foi alocada a outro chamado por outro regulador.
- **Passos:**
  1. O sistema rejeita a alocação por conflito de estado[cite: 21].
  2. O sistema atualiza a lista de viaturas disponíveis e notifica o MedicoRegulador para escolher outra unidade[cite: 20].
- **Estado final:** Alocação duplicada evitada; `chamado_id` aguarda nova seleção da `ViaturaUSA`[cite: 13, 22].

## Regras de negócio aplicadas

| Regra | Descrição |
| --- | --- |
| RN-03 | Uma viatura de Suporte Avançado (`USA`) só pode ser despachada com equipe completa habilitada[cite: 20, 22]. |
| RN-04 | É proibida a alocação simultânea da mesma viatura em mais de um despacho ativo. |
| Evento Sensível C5 | O acionamento de suporte avançado exige emissão do evento `SUPORTE_AVANCADO_ACIONADO` para o módulo G6[cite: 15, 16]. |

## Relacionamentos

- `<<extend>>` de `UC-G02-01` (Despachar Viatura) — estende o fluxo de despacho quando ocorre agravamento ou necessidade de recurso avançado complementar durante ou após o atendimento inicial[cite: 20].
- `<<include>>` `UC-G02-05` (Verificar Disponibilidade de Equipe) — obrigatório para validar se há `ViaturaUSA` disponível com tripulação completa antes de confirmar o apoio[cite: 20, 22].

## Dados manipulados

| Entidade | Dado | Origem |
| --- | --- | --- |
| Chamado | `chamado_id`, `prioridade`, `localizacao` | G1 (Contrato C1)[cite: 13] |
| ViaturaUSA | `viatura_id`, `tipo` (`USA`), `estado`, `localizacao_atual` | G2[cite: 13, 21, 22] |
| Equipe | `equipe_id`, `membros`, `status_disponibilidade` | G2[cite: 13] |
| Despacho | `despacho_id`, `chamado_id`, `justificativa`, `eta_local` | G2[cite: 13] |

## Critérios de aceite

- [ ] O sistema permite vincular uma `ViaturaUSA` ao mesmo `chamado_id` de um atendimento em curso[cite: 13, 22].
- [ ] O sistema impede o despacho de `ViaturaUSA` sem equipe completa ou fora do estado `Disponível`[cite: 20, 21, 22].
- [ ] Ao concluir o acionamento, o evento `SUPORTE_AVANCADO_ACIONADO` é emitido para o G6 e `despacho_ativo` (C2) é publicado para o G3[cite: 13, 15, 16].

## Rastreabilidade

| RF | Classe | Sequência | Teste |
| --- | --- | --- | --- |
| RF-G02-03 | `Despacho`, `ViaturaUSA`, `Equipe`[cite: 22] | DS-G02-02 | CT-G02-02 |
| RF-G02-05 | `Viatura`, `ViaturaUSA`, `Despacho`[cite: 22] | DS-G02-02 | CT-G02-03 |
| RF-G02-06 | `Despacho`, `ViaturaUSA`, `Chamado`[cite: 22] | DS-G02-02 | CT-G02-04 |
| RF-G02-08 | `Despacho` | DS-G02-02 | CT-G02-04 |