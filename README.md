# Auditoria NF — Validação Automática de Notas Fiscais com IA

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Matheus%20Marques-blue)](https://www.linkedin.com/in/matheus-marques-marinho)
[![GitHub](https://img.shields.io/badge/GitHub-matheusmmarinho7-black)](https://github.com/matheusmmarinho7)

> Projeto de portfólio inspirado em um problema real de rotina financeira/fiscal, desenvolvido para fins de estudo.

---

Em qualquer empresa que recebe notas fiscais de fornecedores, alguém precisa conferir manualmente se cada nota bate com o pedido de compra: fornecedor certo, valor certo, CNPJ certo, dentro do prazo. É um trabalho repetitivo, sujeito a erro humano, e que consome tempo de quem poderia estar cuidando de outras prioridades.

Esse workflow automatiza essa auditoria de ponta a ponta. A nota chega por e-mail em PDF, a IA extrai os dados, o sistema cruza com o pedido de compra no banco, e o resultado — aprovado, divergente ou bloqueado — é registrado e notificado automaticamente, sem intervenção manual.

---

![Fluxo no n8n](./Captura%20de%20tela%202026-07-03%20234742.png)

---

## Como funciona na prática

Quando uma nota fiscal chega por e-mail:

1. O workflow verifica se o anexo realmente é um PDF legível e se o documento é de fato uma Nota Fiscal Eletrônica (não um contrato, boleto ou outro papel qualquer)
2. Extrai os dados da nota via GPT-4o-mini: emitente, CNPJ, chave de acesso, valor total, data de emissão
3. Valida a chave de acesso e a data de emissão (bloqueia notas com mais de 90 dias)
4. Confirma que o CNPJ do destinatário é o correto — bloqueia notas endereçadas a outra empresa
5. Busca o pedido de compra correspondente no banco (Supabase/Postgres) e compara os valores, com tolerância de R$ 0,10 para diferenças de arredondamento
6. Registra o resultado da auditoria no banco e notifica o canal certo no Slack
7. Marca o e-mail como lido ao final de cada rota, aprovada ou bloqueada

---

## O que o workflow audita

| Situação | Resultado |
|---|---|
| Nota bate com o pedido | Aprovado |
| Diferença de valor até R$ 0,10 | Aprovado com ressalva |
| Diferença de valor acima da tolerância | Divergente |
| Fornecedor sem pedido cadastrado | Bloqueado |
| Nota já processada antes | Duplicata bloqueada |
| Chave de acesso ausente ou inválida | Bloqueado |
| Documento não é uma NF-e | Bloqueado |
| CNPJ do destinatário incorreto | Bloqueado |
| Nota emitida há mais de 90 dias | Bloqueado |
| Falha ao extrair o PDF | Erro registrado, sem travar o fluxo |

---

## Como o workflow lida com falhas

Para garantir que nenhuma nota se perca ou trave o processo, mesmo quando algo sai do esperado:

**Always Output Data no Supabase** — mesmo quando a busca por pedidos não retorna nenhuma linha, o node segue adiante com um item vazio em vez de simplesmente parar a execução, permitindo que a lógica de "fornecedor não cadastrado" trate o caso.

**Try-catch na extração da IA** — se a resposta do GPT-4o-mini vier em formato inesperado ou o PDF não puder ser lido, o fluxo captura o erro, registra o motivo no banco e segue para notificação, sem interromper o restante da fila.

**Rotas de erro dedicadas (onError: continueErrorOutput)** — nodes críticos têm saída de erro própria, separada da saída de sucesso, então uma falha pontual não derruba a execução inteira.

**Notificações específicas por tipo de bloqueio** — cada motivo de bloqueio (chave inválida, documento inválido, fornecedor não cadastrado, divergência, duplicata, CNPJ errado, nota antiga, erro de extração) tem sua própria mensagem no Slack, com os dados relevantes daquele caso.

**Error Trigger global** — um workflow separado (`Erro - NF`) que escuta qualquer falha não tratada em qualquer node do fluxo principal. Se algo inesperado acontecer fora das rotas de erro já previstas, um alerta é enviado automaticamente para o Slack com o nome do workflow, o node afetado e o link direto para a execução com erro.

**Marcação de e-mail como lido em toda rota** — aprovado ou bloqueado, o e-mail é sempre marcado como processado ao final, evitando reprocessamento duplicado na próxima execução.

---

## Stack

- **n8n** — orquestração do workflow
- **OpenAI GPT-4o-mini** — extração dos dados da nota fiscal a partir do PDF
- **Supabase (Postgres)** — banco de pedidos de compra e log de auditorias
- **Slack** — notificações por tipo de resultado (aprovado, divergente, bloqueado) e alertas de erro global
- **Gmail** — entrada das notas fiscais por e-mail

---

## Como importar no seu n8n

1. Baixe os arquivos `Auditoria NF.json` (fluxo principal) e `Erro - NF.json` (subworkflow de erro global)
2. No n8n, clique em **+** → **Import from file** e importe os dois
3. Configure as credenciais em cada node: Gmail, OpenAI, Supabase/Postgres e Slack
4. Ajuste o CNPJ da empresa destinatária na constante `CNPJ_NOSSA_EMPRESA`, no node de validação — está fixo no código porque o plano de n8n usado neste projeto não libera a feature de Variables
5. Ajuste os canais do Slack em cada node de notificação
6. No workflow `Auditoria NF`, vá em **Settings → Error Workflow** e selecione o workflow `Erro - NF`
7. Ative os dois workflows

> Empresas com CNPJ por filial devem confirmar qual filial de fato recebe as notas fiscais antes de usar este workflow em produção — o CNPJ usado aqui é um exemplo fixo para fins de demonstração.
