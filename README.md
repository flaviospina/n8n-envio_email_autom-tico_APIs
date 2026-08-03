# n8n - Envio automático de e-mails CECAPE (APIs)

Workflow unificado do n8n que envia, **turma a turma**, os e-mails de aviso do dia
seguinte para **alunos** e, em seguida, para os **formadores** da mesma turma.

Arquivo do workflow: `CECAPE-envio-unificado-alunos-formadores-diario-QR.json`
(importe-o no n8n em *Workflows → Import from File*).

## Como funciona

1. **Config - Variáveis**: nó inicial com as variáveis de teste/execução (veja abaixo).
2. **Preparar Data Alvo**: calcula a data-alvo somando `DIAS_A_FRENTE` dias à
   data atual (`1` = dia seguinte). Regra de fim de semana mantida: se a
   data-alvo for **sábado**, a **segunda-feira** também entra como data-alvo.
3. **Agenda mensal** (planilha `11_isrdRYiew08xo7rZyBqx1YkvyfR7IRmrHui37x768`):
   lê a(s) aba(s) do(s) mês(es)-alvo e guarda os cursos do dia seguinte
   (sala, horário, título, formadores etc.).
4. **Diários**: lista as planilhas da pasta do Drive
   `1-qF8Yh0oIXE8VJwypPjFkcjET1YCVmya` cujo nome segue o padrão
   `AAAA MM DD NOME DO CURSO DIÁRIO QR FORMAÇÃO`.
5. Para cada diário, lê a aba **FREQUÊNCIA**, linha 2 **a partir de L2**, e
   percorre as células comparando com a(s) data(s)-alvo. Ao encontrar a data
   sentinela **01/01/2026**, para de percorrer.
6. Para as turmas com aula na data-alvo, lê os alunos — **coluna C = nome**,
   **coluna D = e-mail**, a partir da **linha 3** — e monta **uma fila única e
   ordenada de envios**: alunos do curso 1 → formadores do curso 1 → alunos do
   curso 2 → formadores do curso 2 → ... (formadores vêm do campo FORMADORES da
   agenda mensal, formato `Nome - email`).
7. Um **único loop** percorre a fila e dispara cada e-mail na ordem (20 s de
   intervalo após e-mail de aluno, 5 s após e-mail de formador). Não há loops
   aninhados: o *Split In Batches* aninhado tem bug conhecido no n8n em que o
   loop interno só roda na primeira iteração do externo
   (github.com/n8n-io/n8n/issues/23670) — por isso a fila única garante que
   TODAS as turmas do dia sejam processadas até o último formador do último
   curso.
8. Gatilhos: manual e agendado (todos os dias às 09:00, fuso America/Sao_Paulo).

## Variáveis de teste (nó "Config - Variáveis")

| Variável | Valores | Descrição |
|---|---|---|
| `MODO_TESTE` | `true` / `false` | `true`: nenhum e-mail vai para alunos/formadores; tudo é redirecionado para `EMAIL_TESTE` e o assunto indica o destinatário real (`[TESTE p/ ...]`). `false`: envio real. |
| `EMAIL_TESTE` | e-mail | Caixa que recebe os envios em modo teste. |
| `DIAS_A_FRENTE` | inteiro >= 0 | Quantos dias após a data atual consultar: `1` = dia seguinte (produção), `0` = hoje, `2` = depois de amanhã etc. As regras de fim de semana continuam valendo sobre a data resultante. |
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
