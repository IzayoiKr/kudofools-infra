# Operations

## OpenBao re-seal

After any pod restart (node reboot, OOM, etc.), OpenBao reseals:

```bash
kubectl exec openbao-0 -- bao operator unseal $(jq -r '.unseal_keys_hex[0]' ~/.bao-keys.json)
kubectl exec openbao-0 -- bao operator unseal $(jq -r '.unseal_keys_hex[1]' ~/.bao-keys.json)
kubectl exec openbao-0 -- bao operator unseal $(jq -r '.unseal_keys_hex[2]' ~/.bao-keys.json)
```

OpenBao's container has `readOnlyRootFilesystem: true`, so `bao login` cannot persist the token. Pass the root token via `BAO_TOKEN` env var instead:

```bash
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao <command>"
```


## Flux health

```bash
flux get kustomizations
flux get helmreleases -A
```

## Rotating secrets

All secrets are stored in OpenBao and synced by ESO. After updating OpenBao, force ESO to sync (see below).

All commands require the OpenBao root token:

```bash
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)
```

### Registry password

The htpasswd entry in OpenBao and Woodpecker's `REGISTRY_PASSWORD` must stay in sync.

```bash
NEW_PASS=$(openssl rand -base64 32)
echo "Plain-text password (update in Woodpecker UI): $NEW_PASS"

HTPASSWD=$(htpasswd -Bbn admin "$NEW_PASS")
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/registry/auth auth.htpasswd='$HTPASSWD'"
```

The registry reads the htpasswd file on every request — no restart needed. Verify auth works:

```bash
kubectl run auth-check --image=alpine:3.21 --rm -it --restart=Never -n woodpecker-pipelines -- sh -c "
  apk add --no-cache curl
  curl -s -u 'admin:$NEW_PASS' 'http://registry-service.default.svc:5000/v2/_catalog'
"
```

### Woodpecker agent secret

```bash
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/woodpecker/secrets WOODPECKER_AGENT_SECRET=<new-value>"
```

### Forgejo secrets

```bash
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/forgejo/secrets LFS_JWT_SECRET=<new-value>"
```

### Cloudflared tunnel token

```bash
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/cloudflared/credentials credentials.json='$(cat /path/to/new/credentials.json)'"
```

## Regenerate Forgejo OAuth

Generate new credentials in Forgejo UI (Settings → Applications → OAuth2), then update OpenBao:

```bash
kubectl exec openbao-0 -- sh -c "BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/woodpecker/secrets \
  WOODPECKER_FORGEJO_CLIENT=<new-client-id> \
  WOODPECKER_FORGEJO_SECRET=<new-client-secret>"
```

## Force ESO sync

ESO refreshes secrets every 1h by default. Force an immediate sync per secret:

```bash
kubectl annotate externalsecret -n default <name> force-sync=$(date +%s) --overwrite
```

Verify the Kubernetes secret was updated:

```bash
kubectl get secret registry-auth -o jsonpath='{.data.auth\.htpasswd}' | base64 -d
```

## Security

### OpenBao exposed via internet

OpenBao UI is accessible at `openbao.kudofools.dev` through Cloudflare. Protections in place:

- **Authentication**: `userpass` auth with dedicated UI user (not root token)
- **Rate limiting**: Traefik middleware (100 req/10s per IP) via `Cf-Connecting-IP`
- **Security headers**: HSTS, XSS protection, nosniff, strict referrer policy
- **Audit log**: All API requests logged to `/tmp/audit.log` (checked via `kubectl exec openbao-0 -- cat /tmp/audit.log`)
- **TLS**: Cloudflare edge terminates TLS; internal traffic is plaintext on cluster network

For additional protection, consider [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/applications/) as an extra auth layer in front of the tunnel.

## Known issues

### `curlimages/curl` DNS resolution fails in-cluster

`curlimages/curl` images after v7.77 have DNS resolution problems in Kubernetes due to Alpine's musl libc resolver interacting badly with `ndots:5` and search domains in `/etc/resolv.conf`. The symptom is `curl: (6) Could not resolve host`.

Use `alpine:3.21` + `apk add curl` instead, or use `curlimages/curl:7.77.0`.
