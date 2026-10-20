# Lab 4 · Multi-service app with Compose

GitHub user: ronysunny1 · Reference files used (Lab 2 fallback): yes

<!-- Keep every heading as it is: the grader finds the sections by name. Fill in every blank (four underscores) and your GitHub user name: check.sh writeup flags anything left. -->

## 1 · Architecture

```mermaid
flowchart LR
  B["Browser on your laptop"]
  subgraph front["network: front"]
    W["web: nginx :"80"]
  end
  A["api :5000 · networks: front, back"]
  subgraph back["network: back"]
    D[("db :5432")]
  end
  V[("volume pgdata")]
  B -->|"HTTP localhost:8080"| W
  W -->|"HTTP api:5000"| A
  A -->|"PostgreSQL db:5432"| D
  D --- V
```

Which arrow crosses from your laptop into Docker: B --> W

## 2 · Predictions and what happened

Part 1: fill in "I predicted" (the letter and a one-line reason) before you write any YAML. Part 3: fill in the rest. A wrong prediction that you explain earns full marks.

| # | I predicted | What I saw (command -> the deciding output line) | Match? | If not: why |
|---|---|---|---|---|
| P1 | b) nothing — web is on network front only; db is on back | docker compose exec web getent hosts db -> prints nothing | yes | web and db share no network |
| P2 | b) is refused — port 5432 is not published to localhost | curl -sv telnet://localhost:5432 -> Connection refused | yes | port 5432 is not exposed to the host |
| P3 | a) still there — data is stored in the persistent named volume pgdata | curl localhost:8080/api/todos -> todos still returned | yes | postgres data persists in volume pgdata |
| P4 | c) table is gone too — docker compose down -v deletes the volume | psql select count(*) -> relation "todos" does not exist | yes | down -v wiped the persistent volume |
| P5 | a) healthy · 500 — /healthz only checks process liveness, not db connectivity | ps shows api Up (healthy), curl returns HTTP/1.1 500 | yes | health check only verifies express process |
| P6 | b) fail: the name isn't found — laptop browser is not in Docker's internal DNS | browser request -> name not resolved | yes | host machine is outside docker internal dns |

## 3 · Backup and restore

```text
(paste the whole output of step 3.3: pg_dump, down -v, the count query, the restore, and the GET with your todo back)
```

Why the restore must run before `api` starts: api runs schema setup/migrations on startup; restoring beforehand prevents race conditions or duplicate key errors.

## 4 · Compose clinic

### c1 (worked example)
- Error line (copied): `connect ECONNREFUSED 127.0.0.1:5432`
- Checkpoint: port, inside `api`
- Fix: `DB_HOST: db`
- Why this fix, and not the tempting one: inside `api`, `localhost` is `api` itself, and nothing listens on 5432 there. Docker's DNS answers `db` with the database container's address. Publishing `db`'s port would "work" by opening a door into the database.
- Hints used: 0

<!-- Copy the block below once for every mission you solved and change cN to c2, c3 … Then delete the cN block itself. -->

### c2
- Committed fix: `lab4/clinic/c2/`
- Error line (copied): `curl: (7) Failed to connect to localhost port 8102`
- Checkpoint (name / path / port / app / config / data): port
- Fix: `8102:80` under `web.ports` in `compose.yaml`
- Why this fix, and not the tempting one: nginx inside the web container listens on port 80, not 3000. Changing the host side to 3000 wouldn't match the clinic check on 8102.
- Hints used: 0

### c3
- Committed fix: `lab4/clinic/c3/`
- Error line (copied): `host not found in upstream "api"`
- Checkpoint (name / path / port / app / config / data): name
- Fix: add `- front` to `api.networks` in `compose.yaml`
- Why this fix, and not the tempting one: web is on network front and cannot resolve api unless api shares the front network. Putting web on the back network would break network isolation.
- Hints used: 0

### c4
- Committed fix: `lab4/clinic/c4/`
- Error line (copied): `GET http://localhost:5000/api/todos net::ERR_CONNECTION_REFUSED`
- Checkpoint (name / path / port / app / config / data): path
- Fix: change `http://localhost:5000/api/todos` to `/api/todos` in `web/app.js`
- Why this fix, and not the tempting one: the browser runs on the user's laptop where localhost:5000 is not exposed. Using a relative path routes through the web reverse proxy on port 8080.
- Hints used: 0

### c5
- Committed fix: `lab4/clinic/c5/`
- Error line (copied): relation "todos" does not exist after restart
- Checkpoint (name / path / port / app / config / data): data
- Fix: add `volumes: - pgdata:/var/lib/postgresql/data` to db in `compose.yaml`
- Why this fix, and not the tempting one: without a named volume mounted to PostgreSQL's data directory, all data is ephemeral and lost when containers stop.
- Hints used: 0

## 5 · Run in one command

```bash
git clone https://github.com/ronysunny1/turiba-devops-labs.git && cd turiba-devops-labs
cp .env.example .env
docker compose up -d --wait
```

## 6 · Answers

### A1
In c1 the API logged `ECONNREFUSED 127.0.0.1:5432`. Which container was `127.0.0.1`, and why does `db` work instead?

The api container itself. Inside a container, 127.0.0.1 (localhost) refers to its own network namespace, not the database container or the host laptop.

### A2
Trace `POST /api/todos` from your browser to PostgreSQL: each hop, with the name and port used. Why does `db` have no `ports:`?

1. Browser (laptop) sends POST to localhost:8080.
2. Docker port mapping forwards port 8080 to web container (nginx) on port 80 over network front.
3. Nginx proxy_pass forwards the request to http://api:5000 over network front.
4. Api container receives request on port 5000 and queries PostgreSQL at db:5432 over network back.

### A3
`docker compose down` kept your todos, and `docker compose down -v` deleted them. Where were they stored, and what exactly does `-v` remove?

The API's /healthz endpoint only checks that the Node.js process itself is alive and answering HTTP requests; it does not ping PostgreSQL. Requests to /api/todos fail with HTTP 500 because that specific route requires a live database connection.

### A4
With `db` stopped, `docker compose ps` showed `api` as healthy, but `/api/todos` answered 500. Explain both. What does `service_healthy` protect against, and what not?

The API's /healthz endpoint only checks that the Node.js process itself is alive and answering HTTP requests; it does not ping PostgreSQL. Requests to /api/todos fail with HTTP 500 because that specific route requires a live database connection.
