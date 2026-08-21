# tag-release-template

Template de GitHub Actions para deploy via tags (SemVer `vX.Y.Z`), com
verificação de CI, backup pré-deploy, dry-run de migrations e smoke test
pós-deploy. Todo dado específico de ambiente (host, caminhos, nomes de banco,
processos PM2, portas) foi extraído para *repository variables* e *secrets* —
para reutilizar em outro projeto com a mesma stack, basta configurar os
valores abaixo. Premissas de stack (monorepo `api`/`web`, MySQL + Prisma,
Vitest) estão hardcoded nos workflows e exigem editá-los — veja "Fora do
escopo das variáveis".

## Workflows

- **`release.yml`** — dispara em push de tag `v[0-9]*.[0-9]*.[0-9]*` (ou
  manualmente via `workflow_dispatch`, útil para rollback/hotfix). Pipeline:
  `resolve` (valida a tag) → `verify` → `backup` → `migration-dryrun` →
  `deploy` → `smoke`.
  - `workflow_dispatch` aceita `tag`, `skip_migrations` (rollback),
    `skip_verify` (rollback de emergência) e `confirm` (precisa ser `yes`).
- **`verify.yml`** — build/typecheck/testes (api + web) e checagem de drift
  de migrations do Prisma. Roda em PRs e é chamado pelo `release.yml`.
- **`_backup.yml`** — workflow reusável (`workflow_call`) com os passos reais
  de backup (banco + envs da VPS). Não roda sozinho.
- **`backup.yml`** — entrypoint do backup: roda no cron diário
  (`0 6 * * *`) e via `workflow_dispatch`, chamando `_backup.yml`. O cron
  fica independente do `release.yml` de propósito: o backup do pipeline de
  release só acontece quando há deploy, então esse schedule garante backup
  mesmo em dias sem release. Horário de cron não aceita `vars`/`secrets`
  (limitação do GitHub Actions) — edite o valor direto no arquivo se quiser
  outro horário.

## Configuração (Settings → Secrets and variables → Actions)

### Secrets

| Secret | Descrição |
|---|---|
| `SSH_PRIVATE_KEY` | Chave privada para acessar a VPS via SSH |
| `SSH_KNOWN_HOSTS` | Entrada de `known_hosts` da VPS |
| `BACKUP_PASSWORD` | Senha usada pelo script de backup de envs |
| `DB_ROOT_PASSWORD` | Senha root do MySQL, usada no dry-run de migrations |

### Variables

| Variable | Exemplo | Uso |
|---|---|---|
| `SSH_TARGET` | `deploy@203.0.113.10` | usuário+host SSH da VPS |
| `NODE_VERSION` | `20` | versão do Node via `nvm use` |
| `SCRIPTS_DIR` | `/home/scripts` | dir dos scripts de backup e dos marcadores de última release |
| `BACKUP_DB_SCRIPT_PATH` | `/home/scripts/project-database-backup.sh` | script remoto de backup do banco |
| `BACKUP_ENVS_SCRIPT_PATH` | `/home/scripts/project-envs-backup.sh` | script remoto de backup de envs |
| `BACKUP_DUMP_DIR` | `/home/backups` | dir onde os dumps `.sql.gz` são gerados |
| `BACKUP_DUMP_PREFIX` | `myproject` | prefixo do arquivo de dump (`prefix-*.sql.gz`) |
| `MIGRATION_DRYRUN_DB` | `migration_dryrun` | nome do banco descartável usado no dry-run |
| `API_DEPLOY_PATH` | `/home/app/htdocs/project/api` | diretório de deploy da API na VPS |
| `WEB_DEPLOY_PATH` | `/home/app/htdocs/project/web` | diretório de deploy do Web na VPS |
| `DEPLOY_GIT_REMOTE` | `git@github.com:org/project.git` | remote git usado no `git checkout` da VPS |
| `PM2_API_PROCESS` | `app` | nome do processo PM2 da API |
| `PM2_WEB_PROCESS` | `web` | nome do processo PM2 do Web |
| `API_HEALTHCHECK_PORT` | `3333` | porta local checada no smoke test da API |
| `WEB_HEALTHCHECK_PORT` | `3000` | porta local checada no smoke test do Web |

## Como usar

### Como criar a tag corretamente

A tag **precisa ser anotada** (`-a`), não leve. O job `resolve` do
`release.yml` checa o tipo da tag (`git cat-file -t`) e rejeita qualquer tag
que não seja do tipo `tag` — uma tag leve (`git tag v1.2.0`, sem `-a`) aponta
direto pro commit e falha essa checagem.

```bash
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
```

- Formato exigido: SemVer estrito `vX.Y.Z` (só números — `v1.2.0-beta` não
  passa na regex `^v[0-9]+\.[0-9]+\.[0-9]+$`).
- O `push` da tag já dispara o `release.yml` automaticamente: `resolve`
  (valida formato, existência e que é anotada) → `verify` → `backup` →
  `migration-dryrun` → `deploy` → `smoke`.
- Se errou a tag (mensagem, versão etc.), não reaproveite/mova — delete e
  recrie:
  ```bash
  git tag -d v1.2.0
  git push origin :refs/tags/v1.2.0
  git tag -a v1.2.0 -m "Release v1.2.0"
  git push origin v1.2.0
  ```

**Rollback / hotfix manual:** aba Actions → `Release` → `Run workflow`,
preenchendo `tag` com a tag alvo, `confirm: yes`, e marcando
`skip_migrations`/`skip_verify` conforme o cenário. O job `deploy` roda sob o
Environment `production`; configure required reviewers em Settings →
Environments → `production` para exigir aprovação humana antes do deploy de
verdade (o `confirm: yes` sozinho é só uma barreira contra clique acidental).

## Sobre o job `migration-dryrun`

Esse job aplica as migrations pendentes contra uma **cópia real dos dados de
produção** (o dump mais recente gerado pelo backup), num banco descartável
descartável, antes de liberar o deploy de verdade. A diferença em relação ao
`migration-check` do `verify.yml` (que já roda em todo PR contra um MySQL
vazio) é que aqui o teste é contra o *shape* real dos dados — pega problemas
que só aparecem com tabela populada (ex.: `NOT NULL` sem default numa coluna
existente, constraint violada por dado real, migration que trava em tabela
grande) e que um banco vazio nunca revelaria.

**Trade-off consciente:** isso acopla o template a MySQL + Prisma + um
backup em disco já configurado na VPS (`BACKUP_DUMP_DIR`/`BACKUP_DUMP_PREFIX`),
o que é bem mais específico do que o resto do pipeline. Decidimos manter o
job mesmo assim — o ganho de segurança (não travar/corromper produção com uma
migration mal comportada) compensa a especificidade. Quem reusar o template
com outro stack de banco/ORM deve remover ou reescrever esse job.

## Fora do escopo das variáveis

O `matrix: project: [api, web]` em `verify.yml` assume um monorepo com essas
duas pastas na raiz. Se o projeto que reusar este template tiver outra
estrutura, edite esse matrix (e os `working-directory` correspondentes)
diretamente no arquivo.
