# WorkAbsence

Sistema web para funcionários solicitarem faltas/folgas e o RH gerenciar aprovações, códigos de empresa e avisos (incluindo demissão).

Protótipo funcional com dois perfis:

- **Funcionário**: calendário de solicitação, motivo, turno, acompanhamento de status e avisos.
- **RH**: cria a empresa, gera códigos, aprova/reprova pedidos, gerencia funcionários e publica avisos.

## Funcionalidades

### Funcionário
- Cadastro com o **código da empresa**; login por e-mail e senha.
- Solicitação de falta por **intervalo de datas** no calendário, com turno e motivo.
- **Falta comum**: exige 30 dias de antecedência.
- **Falta médica**: comprovante obrigatório (JPG/PNG/PDF até 5 MB), **aprovada automaticamente** e sem a regra dos 30 dias.
- Histórico com status (pendente/aprovada/reprovada), observação do RH e cancelamento de pedidos pendentes.
- Menus **Aprovadas** e **Reprovadas** na sidebar, com abas e contadores por status.
- Menu de avisos com indicador de não lido, comunicados e aviso de demissão.

### RH
- Criação da empresa com geração de **dois códigos diferentes**:
  - **código de funcionário**: usado pela equipe no cadastro;
  - **código de RH**: usado por outros membros do RH (não deve ser compartilhado com funcionários).
- Entrada em uma empresa existente pelo código de RH.
- Aprovação/reprovação de solicitações com observação obrigatória na reprovação.
- Menus **Aprovadas** e **Reprovadas** na sidebar, com abas e contadores por status.
- Lista de funcionários com pedidos pendentes e avisos de demissão ativos.
- Publicação de comunicados para todos e avisos de demissão individuais com **15 dias de antecedência** do último dia.
- Regeneração de códigos (somente o responsável pela empresa).

### Interface
- **Tema escuro por padrão**, com botão sol/lua para alternar para o claro.
- A escolha do tema fica salva no navegador (`localStorage`) e é aplicada antes da página pintar, sem flash.
- Calendário com resumo de **início, fim e duração**, destaque do intervalo, botão **Limpar seleção** e suporte ao tema escuro.
- Indicador de desenvolvimento do Next.js oculto (`devIndicators: false`).

## Regras de negócio

| Regra | Detalhe |
| --- | --- |
| Antecedência de falta comum | `data inicial >= hoje + 30 dias` (dias corridos) |
| Falta médica | Comprovante obrigatório; autoaprovada; aceita até 30 dias retroativos e 365 dias futuros |
| Sobreposição | Bloqueia pedidos com datas que se cruzam |
| Reprovação | Exige observação do RH |
| Demissão | Último dia `>= hoje + 15 dias`; bloqueia faltas após o último dia |
| Códigos | Únicos, com dois tipos separados por segurança |
| Escopo | RH acessa somente a própria empresa; funcionário somente os próprios dados |
| Comprovante | Download autenticado: só o dono ou o RH da mesma empresa |

Validações rodam no servidor (não apenas na interface).

## Stack

- [Next.js 15](https://nextjs.org) (App Router) + React 19 + TypeScript
- Tailwind CSS 4
- Prisma ORM 6 + **SQLite local** / **PostgreSQL no Render**
- Sessão JWT em cookie `httpOnly` com `jose` e senhas com `bcryptjs`
- `react-day-picker` para o calendário

> No protótipo a sessão é implementada diretamente com JWT (mais simples de rodar); a estrutura permite migrar para Auth.js depois.

## Rodando localmente

Requisitos: **Node.js 20+** e npm.

```bash
npm install
npx prisma db push
npx prisma db seed
npm run dev
```

Abra <http://localhost:3000>.

Se quiser recriar o banco do zero:

```bash
npm run db:reset
```

### Contas de demonstração

| Perfil | E-mail | Senha |
| --- | --- | --- |
| RH (responsável) | `rh@demo.com` | `123456` |
| Funcionário | `funcionario@demo.com` | `123456` |
| Funcionário | `marina@demo.com` | `123456` |

Códigos da empresa de demonstração:

- Funcionário: `ACME2026`
- RH: `RHACME26`

## Estrutura do projeto

```
app/
  actions/auth.ts          # login, logout e cadastros
  app/                     # área do funcionário
    solicitar/             # calendário + criação de falta
    faltas/                # histórico e cancelamento
    avisos/                # comunicados e demissão
  rh/                      # área do RH
    solicitacoes/          # aprovar/reprovar
    empresa/               # códigos e regeneração
    funcionarios/          # equipe
    avisos/                # publicar comunicados/demissão
  api/comprovante/[id]/    # download autenticado do comprovante
  api/logout/              # encerrar sessão
components/                # layout, calendário, badges, cópia de código
lib/
  dates.ts                 # fuso America/Sao_Paulo e datas UTC
  rules.ts                 # regras de antecedência, médica e demissão
  session.ts               # cookie de sessão e guards de rota
prisma/
  schema.prisma            # modelos
  seed.ts                  # dados de demonstração
scripts/use-postgres.mjs   # converte o schema para PostgreSQL no deploy
render.yaml                # blueprint do Render (banco no Neon)
```

## Deploy no Render + Neon

O banco de produção fica no **Neon**; o `render.yaml` cria apenas o Web Service
Node. O build executa `scripts/use-postgres.mjs`, que troca o provider do Prisma
de `sqlite` para `postgresql`, adiciona `directUrl` e aplica o schema com
`prisma db push`. Nenhuma migration é necessária neste protótipo.

### 1. Criar o banco no Neon

1. Crie um projeto no Neon na região **Ohio (us-east-2)**.
2. Copie as duas connection strings exibidas no painel:
   - **Pooled connection** → será o `DATABASE_URL`;
   - **Direct connection** → será o `DIRECT_URL`.
3. Confirme que as URLs terminam com `?sslmode=require`.

### 2. Criar o serviço no Render

1. Suba o repositório no GitHub.
2. No Render, escolha **New → Blueprint** e selecione o repositório.
3. Confirme o serviço `workabsence`.
4. Quando o Render solicitar, preencha:
   - `DATABASE_URL` com a conexão **pooled** do Neon;
   - `DIRECT_URL` com a conexão **direta** do Neon.
5. `AUTH_SECRET` é gerado automaticamente pelo Render.
6. Após o deploy, acesse a URL gerada e crie a primeira empresa pelo cadastro de RH.

Observações:

- O `prisma db push` roda no build e cria as tabelas automaticamente.
- `prisma db push` é adequado ao protótipo; para evoluir o produto, troque por migrations versionadas.
- Para produção real, configure backup do banco e mantenha o `AUTH_SECRET` em segredo.

## Limitações e próximos passos

- Sem envio de e-mail/push: avisos aparecem apenas no site.
- Sem controle de feriados, fins de semana ou escalas.
- Anexo único por pedido (5 MB).
- Sem fluxo de recuperação de senha.
- Aviso de demissão não desativa a conta automaticamente; apenas bloqueia novas faltas após o último dia.
- Possíveis evoluções: notificações por e-mail, múltiplos anexos, migrations, painel multiempresa para administradores e testes automatizados das regras.
