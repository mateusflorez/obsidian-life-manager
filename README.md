# Life Manager Vault

Bem-vindo ao seu cofre pessoal no Obsidian. Aqui você centraliza finanças, tarefas, investimentos e perfil em um único espaço totalmente automatizado com Dataview, Meta Bind e Templater.

## 📁 Estrutura

- `Landing.md`: hub principal com os botões dos módulos e o painel “Overview” (avatar, nome, métricas de tarefas/finance/investments).
- `Finance.md` e `finance/<ano>/<Mês>.md`: dashboards e notas mensais. Os meses seguem o padrão `finance/YYYY/Month.md`.
- `Todo.md` + `todo/tasks.md`: gerenciador de tarefas com estados persistidos no frontmatter.
- `Investments.md` + `investments/*.md`: controle de aportes por investimento com gráfico de evolução e formulário para atualizar o valor total.
- `profile/`: inclui `stats.md` (dados do usuário) e `pfp.*` (avatar exibido no Landing).
- `templates/`: modelos usados pelos botões Meta Bind (novo mês financeiro, novo investimento, etc.).

## 🚀 Como usar

1. **Landing / Overview**
   - Ajuste `profile/stats.md` com `- name: Seu Nome` e coloque uma foto em `profile/pfp.png|jpg|jpeg|webp|gif`.
   - O painel mostra automaticamente:
     - Total investido no mês (somando aportes com tag de data atual em `investments/`).
     - Tasks pendentes (todo/daily/weekly/monthly) baseadas no estado salvo em `Todo.md`.
     - Saldo financeiro do mês vigente (Receitas – Despesas da nota atual em `finance/`).

2. **Finanças**
   - Use o botão “Novo mês financeiro” em `Finance.md` para gerar o arquivo pelo template `templates/new finance month.md`.
   - Dentro de cada mês, registre linhas no formato `categoria:: valor #tag`. O dashboard lê qualquer categoria para montar tabela e gráficos.

3. **Tarefas**
   - Liste tarefas em `todo/tasks.md` sob os blocos `## todo`, `## Daily`, `## Weekly`, `## Monthly`.
   - `Todo.md` renderiza essas listas e persiste o estado em `todoStatus`, `dailyStatus`, etc. Remover uma linha do arquivo também remove o status salvo.

4. **Investimentos**
   - Cada nota em `investments/` possui `## Movimentações` com linhas `- valor #YYYY-MM-DD (#initial opcional)`.
   - Em `Investments.md`, informe o novo valor total do investimento no formulário; o script calcula a diferença e grava a linha na nota com a data atual.
   - O gráfico (Chart.js) acompanha a evolução dos últimos 12 meses para cada investimento.

5. **Templates**
   - `templates/new finance month.md`: cria a estrutura padrão de despesas/receitas.
   - Outros templates podem ser usados pelos botões Meta Bind (como novos investimentos ou páginas utilitárias).

## ✅ Requisitos

- Obsidian com os plugins: **Dataview**, **Meta Bind**, **Templater** (todos já referenciados nas notas).
- Nome das notas mensais em inglês (`November`, `December`, etc.) para que o painel do Landing localize o arquivo correto via `moment().format("MMMM")`.

## 👋 Boas-vindas

Abra o `Landing.md`, configure seu nome/avatar e comece pelos botões principais:

1. Crie o mês atual com **Finance** (botão “Novo mês financeiro”).
2. Registre tarefas em `todo/tasks.md` e acompanhe o progresso em **To-do**.
3. Cadastre seus investimentos em `investments/` e observe o gráfico em **Investments**.

Pronto! Sua rotina financeira e produtiva agora fica centralizada e sempre atualizada ao abrir o Obsidian. Bons registros! 🧠📈
