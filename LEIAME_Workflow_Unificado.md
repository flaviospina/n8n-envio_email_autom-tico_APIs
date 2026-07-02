# CECAPE — Envio Unificado Alunos e Professores (D+1)

Workflow n8n que unifica os dois fluxos (envio de e-mail + registro na planilha de
controle) em um único processo diário.

**Arquivo:** `CECAPE_Envio_Unificado_Alunos_Professores.json`
(importar no n8n em *Workflows → Import from File*).

---

## Por que a planilha de controle não atualizava (bugs do workflow antigo)

1. **"Sheets - Controle Aluno"** mapeava as colunas com as chaves `A`, `B`, `C`…`G`.
   O nó Google Sheets casa as chaves com os **nomes dos cabeçalhos** da planilha
   ("Data de início do curso", "Nome do curso" etc.). Como `A` não é um cabeçalho,
   **nada era gravado**.
2. **"Sheets - Controle Professor"** estava com o mapeamento **vazio** (`"value": {}`)
   — não gravava coluna nenhuma.
3. **"IF - Data Confere?" estava com as saídas invertidas**: quando a data era
   encontrada na FREQUÊNCIA (true), o fluxo ia para "Sem Data na Freq" (descartava o
   curso); quando NÃO era encontrada (false), tentava montar a fila (vazia).
   Resultado: nenhum registro válido chegava à planilha de controle.

Os três problemas foram corrigidos no workflow novo.

---

## O que o workflow faz (fluxo completo)

1. **Dispara diariamente às 08:00** (Schedule Trigger — horário ajustável) ou
   manualmente (Manual Trigger, para testes).
2. **Calcula o dia seguinte** (D+1) no fuso `America/Sao_Paulo`.
   A constante `OFFSET_DIAS` no nó **"Code - Data Alvo"** permite testar outras
   datas (ex.: `3` = D+3). Em produção deixe `1`.
   No mesmo nó existe a constante **`MODO_TESTE`** (padrão: `true`): enquanto
   estiver `true`, **nenhum e-mail é enviado** — os destinatários passam pelos
   nós de simulação e são registrados na planilha de controle com status
   `Enviado (SIMULAÇÃO)` / `Enviado Formador (SIMULAÇÃO)`. Quando os testes
   estiverem OK, mude para `MODO_TESTE = false` para ativar o envio real.
3. **Localiza a aba do mês** na agenda
   (`11_isrdRYiew08xo7rZyBqx1YkvyfR7IRmrHui37x768`): lê os metadados da planilha e
   escolhe a aba cujo nome contém o mês do dia alvo (ignora acentos/maiúsculas;
   aceita "JULHO", "Julho 2026" etc.). Se o dia seguinte cair no mês que vem
   (execução no último dia do mês), a aba correta é escolhida automaticamente.
4. **Filtra os cursos do dia alvo** pela coluna A (DATA) e lê as colunas A–N
   (DATA, INÍCIO, TÉRMINO, DIA DA SEMANA, ESPAÇO, FORMADORES, TÍTULO, ETAPAS,
   COMPONENTE CURRICULAR, PÚBLICO-ALVO, FORMATO, VAGAS, DIVULGAR, GOOGLE SALA DE AULA).
   A coluna A pode conter **só o dia** (`6` ou `06`), **dia/mês** (`6/7`,
   `06/07`) ou a **data completa** (`06/07/2026`) — todos os formatos são
   reconhecidos, já que a aba é do próprio mês.
5. **Lista os arquivos da pasta** do Drive (`1-qF8Yh0oIXE8VJwypPjFkcjET1YCVmya`),
   incluindo os que estiverem dentro de **subpastas** (1 nível — ex.: pasta do
   dia, MANHÃ/TARDE). O log de execução mostra quantos itens/arquivos foram
   encontrados e, quando um título não casa, lista os nomes disponíveis para
   facilitar o diagnóstico.
   O prefixo `AAAA MM DD ` (data + espaço) do nome do arquivo é ignorado e o
   **TÍTULO** precisa estar **contido** no restante do nome (comparação sem
   acentos, sem pontuação e sem diferenciar maiúsculas). Se o título casar com
   mais de um arquivo, todos seguem no fluxo — a verificação da data na
   FREQUÊNCIA decide qual vale.
6. **Loop por curso** (um de cada vez):
   - Lê a aba **FREQUÊNCIA** do arquivo.
   - **Antes de qualquer coisa**, verifica se o dia alvo está entre as **datas dos
     encontros** — linha 2, a partir da célula **L2** (L2 = data de início do
     curso). Por segurança, se não houver datas na linha 2, verifica também as
     linhas 1 e 3. Datas em número serial ou em texto `dd/mm/aaaa` funcionam.
   - Se a data **não** estiver lá → o curso é ignorado e o loop segue para o próximo.
   - Se estiver → extrai os **alunos**: coluna C (NOME) e coluna D (SCSEDUCA =
     e-mail), a partir da linha 3. Linhas sem e-mail válido são ignoradas.
   - Extrai os **professores** da aba **ID** (D7 = nomes, D8 = e-mails, separados
     por vírgula). Se a aba ID não existir, o curso segue só com os alunos.
7. **Envia os e-mails na ordem exigida**: primeiro **todos os alunos** do curso,
   depois o(s) **professor(es)** — e só então passa ao próximo curso. Repete até
   não haver mais cursos do dia alvo.
8. **Registra cada envio na planilha de controle**
   (`11ecc4hdGNPvRj0wlE3F4UWVrJWNUvCqmlR2bGyPfaLk`), agora com o mapeamento
   correto pelos cabeçalhos:
   | Coluna | Valor |
   |---|---|
   | Data de início do curso | primeira data dos encontros (célula L2) |
   | Nome do curso | TÍTULO da agenda |
   | Nome do(s) formador(es) | nomes da aba ID (ou FORMADORES da agenda) |
   | Data da aula | dia alvo (D+1) |
   | Nome do aluno / E-mail do aluno | destinatário |
   | Status | `Enviado` / `Enviado (Formador)` — ou `... (SIMULAÇÃO)` em modo teste |

---

## O que você precisa fazer após importar

1. **Credencial do Gmail**: os nós **"Gmail - Enviar Aluno"** e
   **"Gmail - Enviar Professor"** foram criados **sem credencial** (não tenho
   acesso à sua). Abra cada um e selecione a credencial Gmail usada no seu
   workflow de envio aos formadores. Enquanto `MODO_TESTE = true`, esses nós
   não são executados, então dá para testar antes mesmo de configurá-los.
2. **Texto dos e-mails**: coloquei um modelo em HTML com título, data, horário,
   espaço, formato e formadores. Se quiser manter exatamente o texto do seu
   workflow antigo de envio, é só colar o assunto/corpo nos dois nós Gmail.
3. **Credenciais Google Sheets/Drive**: os nós já apontam para
   "Google Sheets account 2" e "Google Drive account 2" (as mesmas do seu
   workflow original). Se os IDs não casarem na importação, selecione-as manualmente.
4. **Horário do disparo**: ajuste no nó "Schedule Trigger - Diário" se quiser
   outro horário que não 08:00.
5. **Teste sem esperar o dia certo**: no nó "Code - Data Alvo", mude
   `OFFSET_DIAS` para o número de dias até uma data que tenha curso, execute
   manualmente e confira a planilha de controle (status com "SIMULAÇÃO").
   Quando tudo estiver OK: volte `OFFSET_DIAS` para `1`, mude
   `MODO_TESTE` para `false` e **ative** o workflow.

## Observações

- Os nós Gmail e Sheets de controle estão com *continue on fail*: se um e-mail
  falhar, o loop não trava e os demais destinatários continuam sendo processados.
- A listagem da pasta do Drive traz até 1000 arquivos por execução. Se a pasta
  passar disso, será preciso paginar (me avise que eu incluo).
- A planilha de controle é a mesma do workflow anterior
  (`11ecc4hdGNPvRj0wlE3F4UWVrJWNUvCqmlR2bGyPfaLk`, primeira aba/gid=0), com os
  cabeçalhos: *Data de início do curso, Nome do curso, Nome do(s) formador(es),
  Data da aula, Nome do aluno, E-mail do aluno, Status*.
