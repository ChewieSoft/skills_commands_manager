# Changelog — deploy

Formato: [Semantic Versioning](https://semver.org/)

## [1.4.0] - 2026-03-16

### Corrigido

- Pre-flight: `npx eslint .` corrigido para `npx eslint src/` (alinhado com CI)
- Pre-flight: `yarn vitest run src/test/` corrigido para `yarn test --watchAll=false` (projeto usa Jest, nao Vitest)

### Alterado

- Step 9 reescrito: agora captura o `run-id` do pipeline triggado pelo push
- Novo step 10: monitora o pipeline com `gh run watch <run-id>` ate completar
- Novo step 11: avalia resultado — se falhar, exibe `gh run view --log-failed`; se suceder, reporta com link
- A skill so considera o deploy concluido quando o pipeline terminar com sucesso

### Motivacao

A versao anterior apenas listava os runs recentes sem monitorar o resultado. O usuario precisava verificar manualmente se o pipeline passou. Agora o fluxo e end-to-end.

---

## [1.3.0] - 2026-03-13

### Alterado

- Plugin renomeado de `deploy-staging` para `deploy` (namespace fix)
- Command file renomeado de `deploy-staging.md` para `staging.md`
- Invocação muda de `/deploy-staging:deploy-staging` para `/deploy:staging`
- Preparado para futuros subcomandos (e.g. `/deploy:production`)

---

## [1.2.0] - 2026-03-13

### Adicionado

- Detecção automática de cenário: branch atual `develop` vs feature branch
- Fluxo simplificado quando já em `develop`: sincroniza main com `origin/develop` (ff-only), pusha commits locais e pula direto para verificação de pipeline
- Steps 6-8 preservados como fluxo completo para feature branches

### Motivação

Quando o usuário já está em `develop`, os passos de merge de feature branch são desnecessários. O fluxo simplificado evita checkouts e merges redundantes.

---

## [1.1.0] - 2026-03-13

### Adicionado

- Passo pre-flight: `eslint --max-warnings 0`, `tsc --noEmit`, `vitest run` antes do push
- Aborta o fluxo se qualquer verificação local falhar, evitando falhas no pipeline remoto

### Motivação

Deploy `23060872731` falhou por warning ESLint (`react-refresh/only-export-components`) que teria sido pego localmente.

---

## [1.0.0] - 2026-03-13

### Adicionado

- Workflow completo: verificar working tree → fetch → sincronizar main com develop → merge feature → push develop → verificar pipeline
- Sincronização automática de `main` com `origin/develop` via fast-forward
- Verificação de pipeline via `gh run list`
- Notas sobre CD staging (GHCR `:staging`, self-hosted runner)
