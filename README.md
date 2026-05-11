# AixBOMS × Grafana — viability test

A pre-configured Grafana stack to evaluate whether **Grafana + the Infinity
REST datasource** is a viable dashboarding option for AixBOMS data.

What's inside:

- `docker-compose.yml` — runs one Grafana container with the Infinity plugin
  installed automatically.
- `provisioning/datasources/aixboms.yaml` — creates a datasource called
  **AixBOMS REST** pointing at `http://host.docker.internal:8080` (i.e. the
  AixBOMS server running on your machine).
- `provisioning/dashboards/provider.yaml` — tells Grafana to load any JSON
  dashboards it finds in `./dashboards`.
- `dashboards/pollen-count.json` — a sample dashboard that POSTs queries to
  `/aixboms/rest/sensordata/grafana/query`, which returns a flat JSON array
  Grafana can consume with minimal configuration.

You should not need to edit any Grafana settings by hand to get the first
panel rendering — just bring AixBOMS up, start the container, and open the
dashboard.

---

## 0. Prerequisites

1. **Docker Desktop** installed and running.
   - macOS: <https://www.docker.com/products/docker-desktop/>
   - Windows: same link, choose the WSL2 backend during install.
   - Linux: install `docker` + `docker compose` from your distro's packages.
2. **AixBOMS** running locally and reachable at
   `http://localhost:8080/aixboms/rest/sensordata/grafana/query`.
   - Test from a terminal first:
     ```bash
     curl -X POST http://localhost:8080/aixboms/rest/sensordata/grafana/query \
       -H "Content-Type: application/json" \
       -d '{
         "aggregateFunctions": ["AVG","MAX","MIN","COUNT"],
         "from": { "time": 1514761200000 },
         "aggregationWindow": { "timeUnit": "HOUR", "duration": 12 },
         "sensors": ["pollen_count"]
       }'
     ```
   - The response should be a flat JSON array like:
     ```json
     [
       { "time": 1514761200000, "sensor": "pollen_count", "AVG": 42.3, "MAX": 89.0, "MIN": 12.0, "COUNT": 1.0 },
       ...
     ]
     ```
   - If this returns JSON, you're good. If not, fix AixBOMS access first —
     the rest of this guide depends on it.

> **Why `host.docker.internal` and not `localhost`?**
> Inside a Docker container, `localhost` means *the container itself*, not
> your machine. Docker provides the name `host.docker.internal` to refer to
> the host. The compose file already wires this up on all three platforms.

---

## 1. Start Grafana

From inside this `grafana/` folder:

```bash
docker compose up -d
```

The first time, Docker will:

1. Download the Grafana image (~250 MB).
2. Start the container.
3. Install the Infinity plugin (visible in the logs).
4. Apply the provisioning (datasource + dashboard).

Watch the startup logs to make sure nothing exploded:

```bash
docker compose logs -f grafana
```

Press `Ctrl-C` to stop tailing the logs (the container keeps running).

---

## 2. Log in

Open <http://localhost:3010> in a browser.

- Username: `admin`
- Password: `admin`

Grafana will prompt you to change the password — you can skip this for the
test.

---

## 3. Set the bearer token (if AixBOMS requires authentication)

Skip this section if your local AixBOMS instance accepts unauthenticated requests.

There are two ways to supply the token. **Option A** is the quickest for a local test; **Option B** is reproducible across restarts and useful when sharing the setup with a team.

### Option A — Grafana UI (no restart required)

1. In Grafana, go to **Connections → Data sources** (left nav).
2. Click **AixBOMS REST**.
3. Scroll to the **Auth** section.
4. Under **Bearer token auth**, paste your token into the **Token** field.
5. Click **Save & test** at the bottom.

Grafana stores the token encrypted in its internal database. It survives container restarts as long as you don't wipe the volume (`docker compose down -v`).

### Option B — provisioning file + `.env` (reproducible)

This approach keeps the token out of the YAML file and injects it via an environment variable, which Grafana substitutes at provisioning time.

1. Create a `.env` file next to `docker-compose.yml` (it is already listed in `.gitignore`):

   ```
   AIXBOMS_BEARER_TOKEN=your-token-here
   ```

2. Open [provisioning/datasources/aixboms.yaml](provisioning/datasources/aixboms.yaml) and uncomment the last two lines:

   ```yaml
   secureJsonData:
     bearerToken: $AIXBOMS_BEARER_TOKEN
   ```

3. Restart Grafana so it re-reads the provisioning file:

   ```bash
   docker compose up -d
   ```

To rotate the token later, update `.env` and run `docker compose up -d` again.

---

## 4. Open the sample dashboard

In the left nav: **Dashboards** — find and open **New dashboard** (or whatever
you named it when saving).

You should see a panel showing data returned from
`/aixboms/rest/sensordata/grafana/query`. The panel type defaults to a table;
see section 5 for converting it to a time series chart.

If the panel shows "No data" or an error, jump to
[Troubleshooting](#6-troubleshooting) below.

---

## 5. Display data as a time series chart

The `/sensordata/grafana/query` endpoint returns a flat JSON array that the
Infinity plugin can read with no Rows/Root or column mapping configuration.
The only step needed is telling Grafana that the `time` field is a timestamp.

### Convert the time field

1. In the panel editor, click the **Transform data** tab.
2. Click **Add transformation** → search for **Convert field type**.
3. Set field `time` → convert to **Time**.

### Switch to a time series visualization

1. In the panel editor, click the panel type picker (top right) and select
   **Time series**.
2. Each aggregate field (`AVG`, `MAX`, `MIN`, etc.) becomes a separate line.

### Infinity query settings (for reference)

| Setting | Value |
|---|---|
| Type | JSON |
| Source | URL |
| Method | POST |
| URL | `http://host.docker.internal:8080/aixboms/rest/sensordata/grafana/query` |
| Parser | Backend |
| Rows / Root | *(leave empty)* |
| Columns | *(leave empty — auto-detected)* |

### Save the dashboard back to the repo

Once the panel looks right, export it and overwrite `dashboards/pollen-count.json`:

1. Click **Share** (or gear icon) → **Export as code** → copy the JSON.
2. Paste it into `dashboards/pollen-count.json` in this repo.
3. Run `docker compose down -v && docker compose up -d` to confirm it
   provisions correctly on a clean start.

---

## 6. Troubleshooting

### "Connection refused" / "failed to fetch" in a panel

The container can't reach AixBOMS.

- Confirm AixBOMS is up on the host: `curl http://localhost:8080/...` from
  your normal shell.
- Confirm the container can reach the host: 
  ```bash
  docker exec -it aixboms-grafana curl -v http://host.docker.internal:8080/aixboms/rest/sensordata/grafana/query
  ```
  If this fails on Linux, double-check that `extra_hosts: host-gateway` is
  present in `docker-compose.yml` and that your firewall isn't blocking the
  container's address range.

### Datasource is missing or shows "plugin not found"

Plugin install runs only on container start. If your machine was offline the
first time:

```bash
docker compose down
docker compose up -d
```

### Dashboard panel shows JSON but no chart

The column selectors don't match your response shape. See section 5 above.

### Permission / CORS / TLS issues

Grafana proxies requests *server-side* through the Infinity plugin, so CORS
doesn't apply. If you switch to HTTPS or add auth headers, edit
`provisioning/datasources/aixboms.yaml` and restart with:

```bash
docker compose restart grafana
```

### Start over from scratch

```bash
docker compose down -v   # -v also deletes Grafana's stored state
docker compose up -d
```

---

## 7. Common commands cheat-sheet

| Action                          | Command                            |
| ------------------------------- | ---------------------------------- |
| Start (background)              | `docker compose up -d`             |
| Stop                            | `docker compose down`              |
| Stop **and** wipe state         | `docker compose down -v`           |
| Restart Grafana                 | `docker compose restart grafana`   |
| Tail logs                       | `docker compose logs -f grafana`   |
| Shell inside the container      | `docker exec -it aixboms-grafana bash` |
| List running containers         | `docker ps`                        |

---

## 8. Next steps after the viability test

If the test goes well, the natural follow-ups (in roughly increasing effort):

1. **Add an auth token** to the datasource — see [section 3](#3-set-the-bearer-token-if-aixboms-requires-authentication).
2. **Add a sensor variable** to the dashboard so the panel queries any
   sensor, not just `pollen_count`. This requires a discovery endpoint
   (`GET /api/sensors` or similar) that returns a list — wire it as a
   Query-type variable using the same Infinity datasource.
3. **Move from Infinity → a JSON-API contract** on the AixBOMS side, so
   query UX is built-in instead of customers mapping JSON manually.
4. **Build a branded AixBOMS datasource plugin** (TypeScript Grafana plugin
   SDK) — the productized end state.
