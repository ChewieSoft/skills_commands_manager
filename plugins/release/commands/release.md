---
description: Gera release notes automaticamente a partir do último release e cria um novo GitHub Release via gh CLI.
metadata:
  version: 1.0.0
---

## Entrada do Usuário

```text
$ARGUMENTS
```

Interpretar a entrada:

- **Versão semântica** (ex: `3.0.0`, `2.1.0`): usar como versão do novo release.
- **Vazio**: perguntar ao usuário qual versão usar antes de prosseguir.

---

## Objetivo

Criar um **GitHub Release** completo com release notes detalhadas em **português do Brasil**, cobrindo todas as alterações desde o último release tag existente no repositório.

---

## Restrições

- **NÃO** modificar arquivos do projeto. Este comando é somente leitura + criação de release.
- **NÃO** fazer push de código ou commits.
- **NÃO** inventar informações — tudo deve vir dos dados do git.

---

## Execução — Passo a Passo

### 1. Detectar o nome do projeto

```bash
# Nome do projeto (package.json name ou nome do diretório)
PROJECT_NAME=$(node -p "try{require('./package.json').name}catch{''}" 2>/dev/null || basename "$(git rev-parse --show-toplevel)")
```

Se `PROJECT_NAME` ficar vazio, usar `basename "$(git rev-parse --show-toplevel)"` como fallback.

### 2. Detectar o último release

```bash
# Último tag de release (ordenado por versão semântica)
git tag -l --sort=-version:refname | head -1
```

Se não houver nenhuma tag, informar o usuário e abortar.

Guardar o resultado como `$LAST_TAG`.

### 3. Coletar dados do git

Executar **em paralelo**:

```bash
# Commits desde o último release (sem merges)
git log $LAST_TAG..HEAD --format="%h %s%n%b" --no-merges

# Estatísticas de arquivos alterados
git diff --stat $LAST_TAG..HEAD

# PRs mergeados (commits de merge)
git log $LAST_TAG..HEAD --merges --oneline

# Contribuidores
git log $LAST_TAG..HEAD --format="%aN" --no-merges | sort | uniq

# Total de arquivos e linhas
git diff --shortstat $LAST_TAG..HEAD
```

### 4. Analisar e categorizar os commits

Ler o corpo e o título de cada commit para classificar em categorias:

| Prefixo / Padrão        | Categoria            |
|--------------------------|----------------------|
| `feat:`                  | ✨ Novas Features    |
| `fix:`                   | 🐛 Bug Fixes        |
| `refactor:`              | 🏗️ Refatoração      |
| `docs:`                  | 📚 Documentação      |
| `test:`                  | 🧪 Testes            |
| `perf:`                  | ⚡ Performance       |
| `security` / `harden`   | 🔒 Segurança         |
| `chore:` / `ci:` / `build:` | 📦 Infraestrutura |
| Sem prefixo             | Analisar conteúdo e classificar manualmente |

Para cada **feature** (`feat:`), agrupar por PR/branch de origem quando possível, criando uma subseção com título descritivo.

### 5. Identificar dependências adicionadas/removidas

```bash
# Diferenças no package.json
git diff $LAST_TAG..HEAD -- package.json
```

Analisar o diff para listar dependências adicionadas, removidas e atualizadas.

### 6. Montar o Release Note

Usar **exatamente** este formato (adaptar seções conforme os dados coletados — omitir seções vazias):

````markdown
# 🚀 $PROJECT_NAME v$NEW_VERSION

**Release Date:** $DATA_HOJE
**Full Changelog:** $LAST_TAG...v$NEW_VERSION
**$N files changed** — $INSERTIONS insertions(+), $DELETIONS deletions(-)

---

## ✨ Novas Features

### Título descritivo da feature (#PR — `nome-da-branch`)
- Bullet point descrevendo a alteração com detalhes técnicos relevantes
- Mencionar endpoints, módulos, integrações criados
- Usar **negrito** para termos técnicos importantes

---

## 🏗️ Refatoração

- **Título curto** — descrição da refatoração com contexto

---

## 🔒 Segurança

- **Título curto** — descrição da melhoria de segurança

---

## 🐛 Bug Fixes

- **Identificador (se houver)** — descrição da correção

---

## ⚡ Performance

- **Título curto** — descrição da melhoria

---

## 📚 Documentação

- Listar documentações adicionadas ou atualizadas
- Mencionar arquivos específicos quando relevante

---

## 🧪 Testes

- **N arquivos de teste** adicionados/modificados
- Listar tipos de teste: integração, unitário, e2e
- Mencionar domínios/módulos cobertos

---

## 📦 Dependências

- **Adicionadas:** listar pacotes novos
- **Removidas:** listar pacotes removidos
- **Atualizadas:** listar pacotes com mudança de versão
- Runtime: informações sobre Node.js, TypeScript etc.

---

## Pull Requests incluídos

- #N — título/descrição curta do PR

---

**Contributors:** @usernames
````

### 7. Criar o Release

Imediatamente após montar o release note, criar o GitHub Release **sem pedir confirmação**:

```bash
gh release create v$NEW_VERSION --target main --title "v$NEW_VERSION" --notes "$RELEASE_NOTES"
```

### 8. Exibir resultado

Exibir o release note completo na conversa junto com a URL do release criado.

---

## Regras de Qualidade

1. **Não inventar** — cada item do release note deve ter um commit ou PR correspondente.
2. **Português do Brasil** — todo o texto deve estar em PT-BR.
3. **Detalhes técnicos** — mencionar endpoints, módulos, arquivos e tecnologias específicas.
4. **Agrupamento por feature** — commits relacionados ao mesmo PR/feature devem ser agrupados, não listados individualmente.
5. **Omitir seções vazias** — se não houver bug fixes, não incluir a seção 🐛.
6. **Formato consistente** — seguir exatamente o template acima, incluindo emojis nos headers.
