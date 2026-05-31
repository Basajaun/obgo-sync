# MAINTENANCE.md — fork mambrique (Basajaun/obgo-sync)

> Overlay downstream. NO toca el `CLAUDE.md` de upstream (evita conflictos en merge).
> Lee este fichero ANTES de trabajar en el fork. Proyecto **durmiente**: solo se activa
> cuando el servicio se rompe o upstream cambia el protocolo LiveSync.

## Por qué este fork existe

`obgo` (upstream `jookos/obgo-sync`) es un cliente headless en Go que sincroniza un vault
Obsidian con CouchDB hablando el protocolo de [obsidian-livesync](https://github.com/vrtmrz/obsidian-livesync).
Lo usamos en EDGARBOT para mantener `~/notas` (vault compartido por claude/agy/codex/openclaw)
sincronizado en tiempo real con el CouchDB de `notas-sync.mambrique.com` (db `obsidian-vault`),
SIN correr una GUI de Obsidian.

Forkeado porque el upstream es joven (12★, AI-generated, "use at your own peril"). Si el autor
lo abandona y LiveSync cambia el wire protocol, mantenemos el fork vivo nosotros.

## Misión (scope mínimo — no arrastres contexto del homelab)

1. **Mantener vivo** el daemon `obgo pull -w` en EDGARBOT.
2. **Trackear upstream**: `git fetch upstream && git merge upstream/main` periódico.
3. **Parchear** si LiveSync rompe el protocolo o aparece un bug que nos afecta.

Fuera de scope: todo lo demás de la infra mambrique.

## Contexto técnico necesario

- **Protocolo LiveSync**: CouchDB `_changes` feed (long-poll), documentos chunked, E2EE AES,
  path obfuscation. Ref: https://github.com/vrtmrz/obsidian-livesync
- **Config** (env o `.env` en cwd):
  | Var | Valor mambrique |
  |---|---|
  | `COUCHDB_URL` | `https://<user>:<pass>@notas-sync.mambrique.com/obsidian-vault` |
  | `OBGO_DATA` | `/home/gorka/notas` (en EDGARBOT) o volumen montado (en container) |
  | `E2EE_PASSWORD` | passphrase del plugin LiveSync (NUNCA a git) |
  | `OBGO_PATH_OBFUSCATION` | `auto` |
- **CLI**: `obgo pull [path]` / `push` / `list`. Daemon: `obgo pull -w -v`.
- **Build**: `make build` → `./obgo`. Tests: `make test` (sin Docker), `make test-integration` (necesita `make couchdb`).

## Deploy (NO hecho aún — paso separado)

Dos opciones, **preferida la de container** (consistente con resto del stack vía Arcane):

- **A) Container (preferida)**: `make image` / `Dockerfile` del repo → imagen → añadir al stack en
  `mambrique-docker-repo`, montar `~/notas` como volumen compartido entre apps. Gestionar en Arcane.
- **B) Binario + systemd**: `make build`, `.env` en cwd, unit `obgo-sync.service` con `obgo pull -w`.

Requiere la passphrase E2EE del owner. El servidor CouchDB ya existe (ver
`selfhost/docs/services/obsidian-livesync.md`).

## Mantenimiento rutinario

```bash
cd ~/projects/obgo-sync
git fetch upstream
git log --oneline HEAD..upstream/main   # ¿hay cambios nuevos?
git merge upstream/main                  # integrar
make test                                # validar antes de redeploy
```

## Enlaces

- Upstream: https://github.com/jookos/obgo-sync
- Fork: https://github.com/Basajaun/obgo-sync
- Runbook CouchDB server: `~/projects/selfhost/docs/services/obsidian-livesync.md`
