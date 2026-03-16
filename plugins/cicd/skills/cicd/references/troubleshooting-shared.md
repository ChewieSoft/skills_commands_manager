# Troubleshooting — Cenários Compartilhados (Infra)

Cenários de infraestrutura que se aplicam tanto ao backend quanto ao frontend.

---

## 1. `unauthorized` no GHCR (Self-Hosted Runner)

**Mensagem:**

```text
Error response from daemon: Head "https://ghcr.io/v2/.../manifests/...": unauthorized
```

**Causa:** O job Deploy no self-hosted runner não autenticava no GHCR antes do `docker compose pull`. Cada job do GitHub Actions tem contexto isolado — o login feito no job Build & Push não persiste para o job Deploy.

**Diagnóstico:**

```bash
# Verificar se o deploy job tem step de login antes do pull
grep -A5 "Login to GHCR" .github/workflows/cd-staging.yml
```

**Correção (workflow — recomendada):**

Adicionar `docker/login-action@v3` no job Deploy, antes do `docker compose pull`:

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

Preferir `docker/login-action@v3` sobre `docker login` manual porque:
- **Logout automático** no post-step (limpa credenciais mesmo se o job falhar)
- **Config isolada** por job (evita race condition em `~/.docker/config.json`)
- **Masking de credenciais** nos logs via `@actions/core`

**Correção alternativa (manual no servidor — apenas para debug):**

```bash
echo "$TOKEN" | docker login ghcr.io -u USERNAME --password-stdin
```

---

## 2. `network declared as external, but could not be found`

**Mensagem:**

```text
network <nome> declared as external, but could not be found
```

**Causa:** O nome da rede Docker no `docker-compose.yml` (via secret `NGINX_NETWORK_NAME`) não corresponde ao nome real da rede criada pelo nginx-proxy. O nome depende do diretório do compose (ex: `nginx-proxy_default`, `proxy_default`).

**Diagnóstico:**

```bash
docker network ls | grep proxy
```

**Correção:**

```bash
# Descobrir o nome correto da rede
docker network ls | grep proxy

# Atualizar o secret com o nome correto
gh secret set NGINX_NETWORK_NAME --env staging --body "nome_correto_da_rede"
```

---

## 3. `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`

**Sintoma:** O browser retorna `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`. O `curl -svk` mostra `sslv3 alert handshake failure`.

**Causa:** O nginx-proxy responde na porta 443, mas **não tem certificado válido** para o domínio — serve o certificado default (auto-assinado). O Let's Encrypt **não emitiu** o certificado porque o DNS não aponta para o IP do servidor.

**Diagnóstico:**

```bash
# 1. Verificar DNS
dig dominio +short
# Deve resolver para o IP do servidor

# 2. Testar TLS handshake
curl -svk https://dominio 2>&1 | grep -E "SSL|alert|subject"

# 3. Testar porta 80 (HTTP-01 challenge)
curl -sv http://dominio 2>&1 | head -10
```

**Correção:**

1. **DNS não aponta:** Configurar registro A/CNAME para o IP do servidor
2. **acme-companion não rodando:** Verificar se o container `nginx-proxy-acme` está rodando
3. **Porta 80 bloqueada:** Abrir porta 80 no firewall (necessária para HTTP-01 challenge)
4. **Certificado pendente:** Reiniciar o acme-companion e aguardar

---

## 4. Runner Offline / Labels Não Encontrados

**Mensagem:**

```text
No runner matching the specified labels was found
```

**Causa:** Self-hosted runner com label `staging` ou `production` não está online.

**Diagnóstico:**

```bash
# No servidor do runner
sudo systemctl status actions.runner.*

# Reiniciar se necessário
sudo systemctl restart actions.runner.*.service
```

**Monitorar fila:**

```bash
gh run list --status queued
gh run list --status in_progress
```

---

## 5. Concurrency Group Bloqueando Deploys

**Sintoma:** Deploy fica "queued" indefinidamente.

**Causa:** Com `cancel-in-progress: false`, um deploy pendente bloqueia o próximo. Se o runner está offline, a queue pode acumular.

**Diagnóstico:**

```bash
# Listar runs pendentes
gh run list --status queued
gh run list --status in_progress

# Cancelar run bloqueado
gh run cancel <run-id>
```

**Prevenção:** Monitorar via `gh run list` antes de disparar novos deploys.
