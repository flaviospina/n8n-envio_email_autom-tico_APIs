# n8n - Envio automático de e-mails CECAPE (APIs)

Workflow unificado do n8n que envia, **turma a turma**, os e-mails de aviso do dia
seguinte para **alunos** e, em seguida, para os **formadores** da mesma turma.

Arquivo do workflow: `CECAPE-envio-unificado-alunos-formadores-diario-QR.json`
(importe-o no n8n em *Workflows → Import from File*).

## Como funciona

1. **Config - Variáveis**: nó inicial com as variáveis de teste/execução (veja abaixo).
2. **Preparar Data Alvo**: calcula o dia seguinte (ou usa `DATA_SIMULADA`).
   Regra de fim de semana mantida: se o dia seguinte for **sábado**, a
   **segunda-feira** também entra como data-alvo.
3. **Agenda mensal** (planilha `11_isrdRYiew08xo7rZyBqx1YkvyfR7IRmrHui37x768`):
   lê a(s) aba(s) do(s) mês(es)-alvo e guarda os cursos do dia seguinte
   (sala, horário, título, formadores etc.).
4. **Diários**: lista as planilhas da pasta do Drive
   `1-qF8Yh0oIXE8VJwypPjFkcjET1YCVmya` cujo nome segue o padrão
   `AAAA MM DD NOME DO CURSO DIÁRIO QR FORMAÇÃO`.
5. Para **cada diário** (um por vez):
   - Lê a aba **FREQUÊNCIA**, linha 2 **a partir de L2**, e percorre as células
     comparando com a(s) data(s)-alvo. Ao encontrar a data sentinela
     **01/01/2026**, para de percorrer.
   - Se a turma tem aula no dia seguinte: lê os alunos — **coluna C = nome**,
     **coluna D = e-mail**, a partir da **linha 3** — e envia o e-mail de aviso
     a cada aluno (intervalo de 20 s entre envios).
   - Terminados os alunos, envia o e-mail aos **formadores daquela mesma turma**
     (campo FORMADORES da agenda mensal, formato `Nome - email`).
   - Só então passa para o **próximo diário/turma** — resolvendo o problema de
     ter que rodar a automação de novo quando há mais de um curso no mesmo dia.
6. Gatilhos: manual e agendado (todos os dias às 09:00, fuso America/Sao_Paulo).

## Variáveis de teste (nó "Config - Variáveis")

| Variável | Valores | Descrição |
|---|---|---|
| `MODO_TESTE` | `true` / `false` | `true`: nenhum e-mail vai para alunos/formadores; tudo é redirecionado para `EMAIL_TESTE` e o assunto indica o destinatário real (`[TESTE p/ ...]`). `false`: envio real. |
| `EMAIL_TESTE` | e-mail | Caixa que recebe os envios em modo teste. |
| `DATA_SIMULADA` | `''` ou `'DD/MM/AAAA'` | Vazio: usa automaticamente o dia seguinte. Preenchida: força essa data como "dia seguinte" (as regras de fim de semana continuam valendo). |
| `PASTA_DIARIOS_ID` | id do Drive | Pasta onde estão os diários QR. |
| `AGENDA_SHEET_ID` | id da planilha | Agenda mensal com os dados dos cursos/formadores. |
| `DATA_SENTINELA` | `'01/01/2026'` | Data que interrompe a varredura da linha 2 da aba FREQUÊNCIA. |

## Observações

- Os dados de sala/horário/título/formadores do e-mail vêm da **agenda mensal**,
  casando o nome do curso do arquivo do diário com o TÍTULO da agenda
  (comparação tolerante a acentos/maiúsculas). Se o curso não for localizado na
  agenda, o e-mail dos alunos sai com o nome do curso do arquivo e os campos de
  sala/horário vazios, e nenhum formador é notificado para essa turma.
- O controle de "Status = Enviado" na aba *Respostas ao formulário 1* do fluxo
  antigo foi removido, pois a nova origem dos alunos é a aba FREQUÊNCIA
  (somente leitura). A deduplicação de e-mails acontece dentro da execução.
- Credenciais usadas: Google Sheets account 2, Google Drive account 2 e
  Gmail account 2 (as mesmas dos workflows originais).
