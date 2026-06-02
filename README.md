# ik-fiesta-collab-pipelines

An **example build → deploy pipeline** for a Fiesta Online private server: it
turns your **git-tracked JSON content into SHN/txt** with
[ik-fiesta-collab](https://github.com/IkaronClaude/ik-fiesta-collab) (`fiesta`),
then **deploys the whole stack** with Docker Compose over SSH. Clone it, drop in
your content, set a handful of GitHub secrets, and adjust to taste.

```
content/ (JSON tables)  --fiesta build-->  SHN/txt  --scp-->  host server files/9Data
                                                                        |
                                          docker compose up -d  <--------+
   sql · db bridges · login · worldmanager · zones · proxy · api · webapp · patch-server
```

It ties the whole ecosystem together — the BYO game runtime + SQL + proxy from
[ik-fiesta-docker](https://github.com/IkaronClaude/ik-fiesta-docker), plus
[ik-fiesta-api](https://github.com/IkaronClaude/ik-fiesta-api),
[ik-fiesta-sample-webapp](https://github.com/IkaronClaude/ik-fiesta-sample-webapp),
and [ik-fiesta-patch-server](https://github.com/IkaronClaude/ik-fiesta-patch-server).
**No game content lives here** — you bring the JSON project and the host's BYO
exes/DBs.

## How it works

`.github/workflows/deploy.yml` runs two jobs:

1. **build-content** — `docker run … ik-fiesta-collab build --env <env>` against
   `content/` (your `fiesta` project, where `fiesta.json` lives), producing
   the server SHN/txt under `content/build/<env>/`, uploaded as an artifact.
2. **deploy** — over SSH: copy `docker-compose.yml` + the built data to the host,
   unpack the data into the host's `server files/9Data`, write `.env` from your
   secrets, and `docker compose pull && up -d`.

`docker-compose.yml` is the full stack (based on fiesta-docker's example) with a
**web tier** added: `api`, `webapp`, `patch-server`.

## Quick start

1. **Use this as a template** (or clone) and put your fiesta project in `content/`
   (see [`content/README.md`](content/README.md)).
2. On the **deploy host**: install Docker, place your BYO `server files` (exes,
   `Databases/*.bak`, `GamigoZR/`), and make sure the GHCR images are pullable
   (public, or `docker login ghcr.io` — see the workflow's deploy step).
3. Set the **secrets** and **variables** below in the repo settings.
4. Run the **build-and-deploy** workflow manually from the **Actions** tab
   ("Run workflow"). It's manual on purpose — a fresh clone has an empty
   `content/`, so auto-running on push would just fail. Add a `push:` trigger
   yourself once your content lives here and you want CD.

For a local dry run without CI: `cp .env.example .env`, edit it, build your
content with `fiesta build`, drop the output into your `server files/9Data`, then
`docker compose up -d`.

## Configuration

**Secrets** (Settings → Secrets and variables → Actions → *Secrets*):

| Secret | Purpose |
| --- | --- |
| `SSH_HOST` / `SSH_USER` / `SSH_KEY` | Deploy host address, user, and **private** key. |
| `DEPLOY_PATH` | Dir on the host for the compose file + `.env` (e.g. `/srv/fiesta`). |
| `FIESTA_SERVER` | Absolute path to the host's `server files` tree. |
| `PUBLIC_IP` | Address players reach the host on (proxy rewrites to this). |
| `SA_PASSWORD` | Bundled SQL Server sa password (meets complexity policy). |
| `JWT_SECRET` | API JWT signing key (long random). |

**Variables** (*Variables* tab — non-secret):

| Variable | Default | Purpose |
| --- | --- | --- |
| `FIESTA_ENV` | `server` | Which env in your project to build. |
| `API_URL` | — | Public base URL of the api (the webapp's `/config.json`). |
| `CORS_ORIGINS` | — | Allowed CORS origin(s) for the api (your webapp URL). |

`.env.example` documents the full compose surface (SQL tiers, GamigoZR/XOR blobs,
web-tier ports/images). The workflow writes a minimal `.env` from the secrets
above; extend it there as needed.

## Adjust to your setup

- **Build layout** — the workflow tars `content/build/$FIESTA_ENV/`; point it at
  wherever your project's build output lands.
- **Kubernetes instead of Compose** — swap the deploy job for `kubectl apply`
  against fiesta-docker's `example/k8s/` (kubeconfig in a secret).
- **Patches** — add a `fiesta-patch pack` step and publish its output to the dir
  `patch-server` serves (`PATCH_DIR`).

## License

[Apache License 2.0](LICENSE).
