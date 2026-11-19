# Módulo G2 · Despacho Operacional e Frota (SAPH-Clin)

## 1. Visão e Escopo do Módulo (E1)

### 1.1 Problema Resolvido
O módulo de **Despacho Operacional** elimina o atraso e a falha de comunicação na alocação de ambulâncias entre a regulação médica e as equipes de rua. Ele automatiza a sugestão de viaturas (`USB` — Unidade de Suporte Básico e `USA` — Unidade de Suporte Avançado) com base na gravidade clínica (Protocolo de Manchester) e proximidade geográfica, além de rastrear o ciclo de vida operacional da frota em tempo real.

### 1.2 Fronteiras e Contratos
- **Onde começa (Entrada — Contrato C1 `chamado_classificado`):** O módulo inicia ao consumir de forma assíncrona o evento `chamado_classificado` emitido pelo **G1 (Chamado/Triagem)**, contendo `chamado_id`, `prioridade` (`VERMELHO`, `LARANJA`, `AMARELO`, `VERDE`, `AZUL`), `queixa_principal`, `localizacao` e `triagem_em`.
- **Onde termina (Saída — Contrato C2 `despacho_ativo`):** O módulo encerra a etapa de despacho ao publicar o evento `despacho_ativo` para o **G3 (Telemedicina/Prontuário)** contendo `despacho_id`, `chamado_id`, `viatura_id`/`tipo` (`USB` ou `USA`), `equipe_id` e `eta_local`.
- **Auditoria Transversal (Contrato C5 `evento_sensivel`):** Toda ação crítica de alocação ou apoio avançado emite eventos imutáveis (`VIATURA_DESPACHADA` e `SUPORTE_AVANCADO_ACIONADO`) para o **G6 (Auditoria/Compliance)**.

### 1.3 Fora do Escopo
- Atendimento telefônico inicial (TASI) e classificação de risco pelo Protocolo de Manchester (escopo do **G1**).
- Teleorientação médica pré-chegada, preenchimento de ficha pré-hospitalar e solicitação de pré-admissão hospitalar (escopo do **G3**).
- Gestão de disponibilidade de leitos hospitalares (escopo do **G4**) e controle de estoque/reposição de fármacos em maletas (escopo do **G5**).

### 1.4 Premissas e Restrições
- **Falha de Contrato de Entrada:** Caso um evento `chamado_classificado` chegue sem `prioridade` válida, o G2 bloqueia o despacho, mantém o chamado em `Em Triagem` e emite alerta imediato ao Médico Regulador.
- **Acoplamento por Evento:** Nenhuma decisão clínica ou despacho sofre bloqueio síncrono caso módulos consumidores (G3 ou G6) apresentem indisponibilidade temporária.
- **Conformidade Normativa:** Regras de tripulação mínima e suporte avançado seguem as diretrizes do Conselho Federal de Medicina (CFM) e da ANVISA para atendimento pré-hospitalar móvel.

---

## 2. Catálogo de Requisitos (E2)

### 2.1 Requisitos Funcionais (RF)

| ID | Descrição | Ator(es) | Critério de Aceite |
| :--- | :--- | :--- | :--- |
| **RF-G02-01** | **Recepcionar chamado classificado:** O sistema deve consumir o evento `chamado_classificado` (Contrato C1) e inserir a ocorrência na fila de despacho ordenada por gravidade clínica. | Sistema (G1 $\rightarrow$ G2) | Chamados com `prioridade` válida entram na fila; se `prioridade` estiver ausente, o despacho é bloqueado e o Médico Regulador é alertado. |
| **RF-G02-02** | **Recomendar viatura por gravidade e proximidade:** O sistema deve sugerir `USA` para chamados `VERMELHO`/`LARANJA` e `USB` para `AMARELO`/`VERDE`/`AZUL`, ordenando as viaturas disponíveis pela menor distância/tempo até a `localizacao`. | Médico Regulador | A interface exibe automaticamente a lista de viaturas compatíveis no estado `Disponível` ranqueadas por proximidade. |
| **RF-G02-03** | **Despachar viatura e equipe:** O sistema deve permitir que o Médico Regulador vincule uma viatura `Disponível` e sua Equipe de Intervenção ao chamado, gerando um `despacho_id` único. | Médico Regulador | O estado da viatura muda para `Deslocando para Sinistro`, o chamado passa a `Viatura Despachada` e o contrato `despacho_ativo` (C2) é publicado para o G3. |
| **RF-G02-04** | **Registrar aceite e ETA da equipe:** O sistema deve permitir que a Equipe de Intervenção confirme o recebimento do chamado no terminal móvel e registre/atualize o tempo estimado de chegada (`eta_local`). | Equipe de Intervenção | O campo `eta_local` é preenchido e atualizado no evento `despacho_ativo`, registrando o timestamp exato do início do deslocamento. |
| **RF-G02-05** | **Atualizar ciclo de vida da viatura:** O sistema deve controlar e registrar as transições sequenciais de estado da viatura: `Disponível` $\rightarrow$ `Deslocando para Sinistro` $\rightarrow$ `No Local` $\rightarrow$ `Em Trânsito p/ Hospital` $\rightarrow$ `Em Descontaminação/Reposição` $\rightarrow$ `Disponível`. | Equipe de Intervenção | O sistema impede pular etapas obrigatórias do ciclo e bloqueia nova alocação enquanto a viatura não retornar ao estado `Disponível`. |
| **RF-G02-06** | **Acionar Suporte Avançado (USA) em campo:** O sistema deve permitir que uma equipe `USB` em atendimento solicite apoio de uma `USA` devido a agravamento clínico (ex: PCR no local). | Equipe de Intervenção / Médico Regulador | Cria novo despacho vinculado ao mesmo `chamado_id`, aloca uma `USA` disponível e dispara o evento sensível `SUPORTE_AVANCADO_ACIONADO` ao G6. |
| **RF-G02-07** | **Tratar indisponibilidade de frota:** Na ausência de viatura compatível `Disponível`, o sistema deve alertar o Médico Regulador para redirecionar frota de outra base, alterar o tipo de suporte com justificativa ou devolver o caso para teleorientação (G3). | Médico Regulador | Alerta crítico exibido na tela de regulação e registro obrigatório de justificativa textual para qualquer decisão de exceção. |
| **RF-G02-08** | **Emitir eventos sensíveis de auditoria:** O sistema deve publicar registros imutáveis no padrão `evento_sensivel` (Contrato C5) para toda operação de despacho (`VIATURA_DESPACHADA`) ou acionamento de suporte avançado (`SUPORTE_AVANCADO_ACIONADO`). | Sistema (G2 $\rightarrow$ G6) | Payload gerado contendo `evento_id`, `tipo`, `ator_id`/`papel`, `entidade`/`entidade_id`, `ocorrido_em`, `resultado` e `justificativa` (quando aplicável). |

### 2.2 Requisitos Não Funcionais (RNF)

| ID | Categoria | Descrição com Métrica e Alvo |
| :--- | :--- | :--- |
| **RNF-G02-01** | **Desempenho** | O cálculo de proximidade e a listagem de viaturas disponíveis para um chamado de prioridade `VERMELHO` devem ser concluídos em **$\le$ 1,5 segundo** sob carga de até **30 chamados simultâneos**. |
| **RNF-G02-02** | **Confiabilidade** | **100% das tentativas concorrentes** de despacho da mesma viatura por dois médicos reguladores distintos devem ser bloqueadas pelo controle de estado, garantindo **0 despachos duplicados**. |
| **RNF-G02-03** | **Disponibilidade** | Em caso de falha de comunicação com os módulos G3 ou G6, **100% dos despachos locais** devem continuar operando sem bloqueio síncrono, com enfileiramento e retentativa de entrega em até **5 segundos**. |
| **RNF-G02-04** | **Conformidade** | **100% dos despachos e acionamentos de USA** devem gerar evento auditável imutável com `evento_id` único (garantindo idempotência) e timestamp do fato em até **2 segundos** após a confirmação. |
| **RNF-G02-05** | **Usabilidade** | O fluxo completo de seleção da viatura recomendada e confirmação de despacho na tela do Médico Regulador deve exigir no máximo **3 interações (cliques)** após a seleção do chamado triado. |

---

## 3. Integrantes do Grupo
- Tiago Natanael do Vale Rodrigues
- João Miguel dos Santos Barros Barbosa
- João Borges

---

## 4. Estado Atual dos Artefatos
- [x] **E1** · Visão e Escopo do Módulo (`README.md`)
- [x] **E2** · Catálogo de Requisitos — 8 RF e 5 RNF (`README.md`)
- [ ] **E3** · Diagrama de Casos de Uso e Especificações Textuais (`casos-de-uso/`, `diagramas/`)
- [ ] **E4** · Diagrama de Classes de Domínio e Dicionário de Dados (`diagramas/`)
- [ ] **E5** · Diagramas de Sequência, Máquina de Estados e Atividades (`diagramas/`)
- [ ] **E6** · ADRs, Matriz de Rastreabilidade e Casos de Teste (`adr/`, `matriz-rastreabilidade.md`, `testes/`)
- [ ] **E7** · Protótipos de Baixa Fidelidade e Relatório Técnico (`prototipos/`, `README.md`)

---

## 5. Relatório Técnico Curto (E7 — Em Construção)
- **Decisões Arquiteturais:** Adoção de comunicação assíncrona orientada a eventos nos contratos C1, C2 e C5 para evitar que indisponibilidades nos módulos de Telemedicina (G3) ou Auditoria (G6) atrasem o envio de ambulâncias.
- **Dificuldades Encontradas:** Delimitação exata da fronteira entre a triagem clínica (G1) e a escolha operacional da viatura (G2), resolvida mantendo a classificação de risco exclusivamente no G1 e o vínculo de frota no G2.
- **Limitações Conhecidas:** O cálculo de `eta_local` considera inicialmente a distância viária estimada no momento do aceite, sem integração dinâmica com APIs externas de tráfego em tempo real.
- **O que faria diferente com mais tempo:** Modelaria algoritmos de realocação dinâmica de frota (cobertura de áreas descobertas) quando múltiplas viaturas de uma mesma base estiverem em atendimento simultâneo.