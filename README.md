# arcane-test

POC of [Arcane](https://getarcane.app/docs/get-started/installation), a web UI for managing Docker and Compose stacks, simulating a local infrastructure with a few apps.

## Structure

```
.
├── compose.yaml          # Arcane
├── .env.example          # Arcane variables (copy to .env)
└── projects/             # Arcane's PROJECTS_DIRECTORY
    ├── web/              # nginx + whoami
    ├── database/         # postgres + adminer
    └── monitoring/       # gatus
        └── gatus/config/ # one YAML per app, read recursively
            ├── arcane.yaml
            ├── web/{nginx,whoami}.yaml
            └── database/adminer.yaml
```

Each folder in `projects/` is a Compose stack that shows up in Arcane's **Projects** tab, where you can start, stop and edit it.

## Getting started

```bash
echo "ENCRYPTION_KEY=$(openssl rand -hex 32)" > .env
echo "PROJECTS_DIR=$PWD/projects" >> .env   # absolute path, run from the repo root
docker compose up -d
```

Open http://localhost:3552. Default login: `arcane` / `arcane-admin` (you'll be asked to change it on first login).

Then start the stacks from the UI (Projects → stack → Deploy) or from the CLI:

```bash
docker compose -f projects/web/compose.yaml up -d
```

## Apps and ports

| Stack      | Service     | URL                     |
|------------|-------------|-------------------------|
| —          | Arcane      | http://localhost:3552   |
| web        | nginx       | http://localhost:8081   |
| web        | whoami      | http://localhost:8082   |
| database   | adminer     | http://localhost:8084 (server `postgres`, user/password/db `poc`) |
| monitoring | gatus       | http://localhost:8086 (endpoints in `gatus/config/`) |

Inside containers, use `http://host.docker.internal:<port>` to reach apps in other stacks. `localhost` points to the container itself, and each stack has its own network.

## Notes

- Arcane accesses Docker through `/var/run/docker.sock`, meaning it has full control over the host's Docker. Use it locally only.
- `projects/` is mounted into Arcane at the **same absolute path** as on the host (`PROJECTS_DIR`). This way, relative bind mounts in the stacks (like `./gatus/config`) resolve to the right place when deploying through Arcane too.
- `projects/.env.global` is created by Arcane (variables shared by all stacks) and is kept out of git.

## Gatus: adding a monitor

Create a YAML file at `projects/monitoring/gatus/config/<stack>/<app>.yaml`:

```yaml
endpoints:
  - name: my-api
    group: my-stack
    url: http://host.docker.internal:9999/health
    interval: 30s
    conditions: ['[STATUS] == 200']
```

Gatus detects the change and reloads on its own within ~30s, with no restart and without touching the other files.

## Tearing down

```bash
docker compose down            # stops Arcane (keeps the data)
docker compose down -v         # also removes the Arcane volume
```
