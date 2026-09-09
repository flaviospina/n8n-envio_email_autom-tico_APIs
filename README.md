# n8n - Envio automático de e-mails CECAPE (APIs)

Workflow unificado do n8n que envia, **turma a turma**, os e-mails de aviso do dia
seguinte para **alunos** e, em seguida, para os **formadores** da mesma turma.

Arquivo do workflow: `CECAPE-envio-unificado-alunos-formadores-diario-QR.json`
(importe-o no n8n em *Workflows → Import from File*).

**Fonte única de dados: a pasta dos diários QR no Google Drive.**
A agenda mensal (`11_isrdRY...`) foi removida do fluxo por não ser confiável.

## Como funciona

1. **Config - Variáveis**: nó inicial com as variáveis de teste/execução (veja abaixo).
2. **Preparar Data Alvo**: calcula a data-alvo somando `DIAS_A_FRENTE` dias à
   data atual (`1` = dia seguinte). Regra de fim de semana mantida: se a
   data-alvo for **sábado**, a **segunda-feira** também entra como data-alvo.
3. **Diários**: lista as planilhas da pasta do Drive
   `1-qF8Yh0oIXE8VJwypPjFkcjET1YCVmya` cujo nome contém
   `DIÁRIO QR FORMAÇÃO` (padrão `AAAA MM DD NOME DO CURSO DIÁRIO QR FORMAÇÃO`).
4. Para cada diário, lê a aba **ID** (uma chamada `batchGet` por arquivo):

   | Célula(s) | Conteúdo |
   |---|---|
   | `D4` | Nome do curso |
   | `E4` | Carga horária |
   | `F4` | Modalidade |
   | `D6` | Data de início do curso |
   | `D7` | Nome(s) do(s) formador(es), separados por vírgula |
   | `D8` | E-mail(s) do(s) formador(es) |
   | `B21:B25` | Datas das aulas |
   | `D21:D25` | Horário da aula (mesma linha da data) |
   | `F21:F25` | Sala da aula (mesma linha da data) |

5. **Validação da data da aula (dupla)**: a turma entra no envio se a
   data-alvo aparecer na **linha 2 da aba FREQUÊNCIA** (varrida a partir da
   célula configurada em `CELULA_INICIO_DATAS`, padrão `L2`, parando na
   `DATA_SENTINELA` `01/01/2026`) **ou** nas datas de `ID!B21:B25` — que
   passarão a ser preenchidas a partir de outubro; quando isso acontecer,
   nada precisa mudar no workflow.
   O **horário** e a **sala** vêm da linha de `B21:F25` cuja data casou;
   enquanto essas datas não existirem, usa-se a primeira linha preenchida da
   grade. Sala vazia → o e-mail sai com o aviso *"A sala será orientada no
   dia do curso, ao chegar ao prédio."* (texto editável na variável
   `AVISO_SALA`).
6. **Alunos**: aba **FREQUÊNCIA** do mesmo diário — **coluna C = nome**,
   **coluna D = e-mail**, a partir da **linha 3**. Linhas sem e-mail válido são
   ignoradas; e-mails repetidos na turma são deduplicados.
7. **Formadores**: nomes de `D7` pareados na ordem com os e-mails de `D8`
   (separadores aceitos: vírgula, ponto-e-vírgula, quebra de linha; nos e-mails
   também espaço). E-mail sem nome correspondente usa o próprio e-mail.
8. Monta **uma fila única e ordenada**: alunos do curso 1 → formadores do
   curso 1 → alunos do curso 2 → formadores do curso 2 → ...
9. Um **único loop** percorre a fila e dispara cada e-mail na ordem (10 s de
   intervalo após e-mail de aluno, 5 s após e-mail de formador). Não há loops
   aninhados: o *Split In Batches* aninhado tem bug conhecido no n8n em que o
   loop interno só roda na primeira iteração do externo
   (github.com/n8n-io/n8n/issues/23670) — a fila única garante que TODAS as
   turmas do dia sejam processadas até o último formador do último curso.
10. Gatilhos: manual e agendado (todos os dias às 09:00 — confira se o fuso do
    workflow no n8n está em America/Sao_Paulo).

## Variáveis (nó "Config - Variáveis")

| Variável | Valores | Descrição |
|---|---|---|
| `MODO_TESTE` | `true` / `false` | `true`: nenhum e-mail vai para alunos/formadores; tudo é redirecionado para `EMAIL_TESTE` e o assunto indica o destinatário real (`[TESTE p/ ...]`). `false`: envio real. |
| `EMAIL_TESTE` | e-mail | Caixa que recebe os envios em modo teste. |
| `DIAS_A_FRENTE` | inteiro >= 0 | Quantos dias após a data atual consultar: `1` = dia seguinte (produção), `0` = hoje, `2` = depois de amanhã etc. A regra sábado→segunda vale sobre a data resultante. |
| `PASTA_DIARIOS_ID` | id do Drive | Pasta onde estão os diários QR. |
| `AVISO_SALA` | texto | Mensagem usada no lugar da sala quando `F21:F25` está vazio. |
| `CELULA_INICIO_DATAS` | célula (ex.: `L2`) | Onde começam as datas na linha 2 da aba FREQUÊNCIA. |
| `DATA_SENTINELA` | `'01/01/2026'` | Data que encerra a varredura da linha 2 da FREQUÊNCIA. |

## Campos exibidos nos e-mails

Data da aula, Sala/Local (ou aviso), Horário, Curso, Formador(es) (só no e-mail
do aluno), Carga horária, Modalidade e Início do curso — todos vindos da aba ID
do próprio diário. Os campos antigos da agenda (Etapas, Google Sala de Aula)
saíram do template por não existirem na nova fonte.

## Observações

- A aba **ID** e a aba **FREQUÊNCIA** precisam existir com esses nomes exatos em
  cada diário; arquivo sem a aba ID é simplesmente ignorado no dia.
- Se `D4` estiver vazio, o nome do curso é extraído do nome do arquivo.
- Não há trava de reenvio: executar manualmente após o disparo das 09:00
  reenvia os e-mails do dia.
- Credenciais: Google Sheets account 2, Google Drive account 2 e Gmail
  account 2 (as mesmas dos workflows originais).
