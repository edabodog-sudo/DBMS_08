# DBMS_09 – From Container to Docker Compose

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_09>  
**Prerequisites:** DBMS_01 – DBMS_07, Lecture 09  
**Duration:** 120 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Explain the difference between a **container** and a **virtual machine**
- Start, inspect, and remove Docker **containers** and **images**
- Write a minimal **Dockerfile** and build a custom image from it
- Demonstrate the **ephemeral** nature of container storage
- Attach a **named volume** to a PostgreSQL container and verify data persists
  across container restarts
- Explain why running two services in one container is an anti-pattern
- Connect two containers via a **custom bridge network** using container names
  as hostnames
- Describe all key sections of a **docker-compose.yml** file
- Start a multi-service application with `docker compose up` and take it down
  cleanly
- Use a **`.env` file** to keep credentials out of `docker-compose.yml`
- Explain what a **Multi-Stage Build** is and why it reduces image size
- Apply the **Principle of Least Privilege** by running containers as a
  non-root user
- Use **init scripts** to initialise a PostgreSQL database on first start

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between a Docker image and a running container?
- Why does deleting a container without a volume lose all data written to it?
- Why can `host="localhost"` inside one container not reach a second container?
- What does `docker compose down -v` do that `docker compose down` does not?
- Why should credentials never appear directly in `docker-compose.yml`?

---

## Prerequisites Check

You need Docker and Docker Compose installed.

```bash
docker --version
docker compose version
```

> Both commands should succeed and show version numbers.

> **Screenshot 1:** Take a screenshot showing both version outputs.
>
> `[insert screenshot]`
> <img width="291" height="53" alt="Screenshot 1 juste" src="https://github.com/user-attachments/assets/91402134-f072-4417-bded-f3d863b74ff7" />


---

## 1 – Hello Container

### Step 1 – Run the hello-world Image

```bash
docker run hello-world
```

> Docker pulls the image from Docker Hub (first run only), starts a container,
> prints a message, and exits.

List all containers, including stopped ones:

```bash
docker ps -a
docker images
```

> **Screenshot 2:** Take a screenshot showing `docker ps -a` and
> `docker images` output.
>
> `[insert screenshot]`
> <img width="438" height="96" alt="Screenshot 2 juste" src="https://github.com/user-attachments/assets/e3b57cda-fa53-49aa-b235-b8a5ed907f60" />


### Step 2 – Run an nginx Webserver

```bash
docker run -d -p 8080:80 --name webserver nginx
curl http://localhost:8080
docker logs webserver
docker exec -it webserver bash
```

Inside the container shell, inspect the running process and exit:

```bash
ps aux
exit
```

Stop and remove the container:

```bash
docker stop webserver && docker rm webserver
docker system df
```

### Questions for Section 1

**Question 1.1:** The flag `-d` starts the container in detached mode.
What happens without `-d`, and why is detached mode useful for a web server?

> *Your answer:*Without -d, the container runs in the foreground, and its logs take over your terminal — you can’t type other commands until you stop it.

Detached mode is useful for a web server because it keeps the server running in the background, letting you close the terminal, run other commands, or manage multiple services without interrupting the running application.

**Question 1.2:** `-p 8080:80` maps host port 8080 to container port 80.
Which port is the application actually listening on inside the container?
What would `-p 9000:80` change?

> *Your answer:*Inside the container, the application is actually listening on port 80 — that’s the container’s internal port.

If you change it to -p 9000:80, the app still listens on port 80 inside the container, but you would reach it from your host at http://localhost:9000 instead of http://localhost:8080.

---

## 2 – Writing a Dockerfile

### Step 1 – Create the Project Directory

```bash
mkdir ~/dbms09_dockerfile
cd ~/dbms09_dockerfile
git init
git remote add origin git@github.com:<your-username>/dbms09_dockerfile.git
```

### Step 2 – Write the Dockerfile

```bash
vim Dockerfile
```

```dockerfile
FROM debian:13
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

### Step 3 – Build and Run

```bash
docker build -t mein-debian .
docker run -it mein-debian
```

Inside the container:

```bash
curl --version
whoami
exit
```

> **Screenshot 3:** Take a screenshot showing the `docker build` output and
> the commands run inside the container.
>
> `[insert screenshot]`
> <img width="452" height="68" alt="Screenshot 3" src="https://github.com/user-attachments/assets/a5f97a61-014e-4bb3-a688-0eeb58586724" /><img width="457" height="116" alt="Screenshot 3a" src="https://github.com/user-attachments/assets/6043cb90-f957-4f0a-9a4e-71fed83eefd8" />



### Step 4 – Commit

```bash
git add Dockerfile
git commit -m "feat: minimal Debian image with curl"
git push -u origin main
```

### Questions for Section 2

**Question 2.1:** Why does the `RUN` instruction combine `apt-get update`,
`apt-get install`, and `rm -rf /var/lib/apt/lists/*` in a single line?
What would happen to the image size if these were three separate `RUN` lines?

> *Your answer:*They are combined into one RUN line to keep all temporary APT files inside a single image layer. When you run apt-get update, apt-get install, and rm -rf /var/lib/apt/lists/* in three separate RUN instructions, Docker creates three layers, and the large APT cache from the first two layers cannot be removed — it stays permanently in the image.

This makes the final image much larger. Combining them into one RUN line ensures the cleanup happens before the layer is saved, keeping the image small and efficient.

**Question 2.2:** `EXPOSE 80` in a Dockerfile does **not** actually open port
80. What does it do, and what is required at `docker run` time to actually
forward a port?

> *Your answer:*EXPOSE 80 is only documentation inside the image — it tells Docker “this container intends to use port 80,” but it does not open or forward anything by itself.

To actually make the port reachable from your host, you must specify a mapping at run time, for example:docker run -p 8080:80 …


---

## 3 – The Persistence Problem

### Step 1 – Start a PostgreSQL Container Without a Volume

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -d postgres:16
```

Connect and create a table:

```bash
docker exec -it pg psql -U postgres
```

Inside `psql`:

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Hallo Docker');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate the Container

```bash
docker stop pg && docker rm pg
docker run --name pg -e POSTGRES_PASSWORD=geheim -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> Expected output: `ERROR: relation "test" does not exist`

> **Screenshot 4:** Take a screenshot showing the error message.
>
> `[insert screenshot]`,
> <img width="587" height="43" alt="Screenshot 4" src="https://github.com/user-attachments/assets/8bdcc88c-90ab-44a7-b0a8-965e806676f0" />


### Questions for Section 3

**Question 3.1:** You stopped and removed the container but the image
`postgres:16` still exists on your machine. Why does recreating a container
from the same image not restore the data?

> *Your answer:*Because a container is ephemeral — when you delete it, all data stored inside its filesystem disappears. The image (postgres:16) only contains the program PostgreSQL, not your database files.
Re‑creating a container from the same image gives you a fresh, empty PostgreSQL instance, because the data was stored in the container’s writable layer, not in the image.

**Question 3.2:** `docker stop` sends SIGTERM and waits for the process to
exit cleanly. `docker kill` sends SIGKILL immediately. Why is `docker stop`
preferred for a database container?

> *Your answer:*docker stop is preferred because it gives PostgreSQL time to shut down cleanly, flush data to disk, close connections, and avoid corruption.
docker kill forces an immediate SIGKILL, which can interrupt writes and leave the database in an inconsistent or damaged state.

---

## 4 – Named Volumes

### Step 1 – Create a Volume and Attach It

```bash
docker volume create pg_data
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Insert data:

```bash
docker exec -it pg psql -U postgres
```

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Daten überleben');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate With the Same Volume

```bash
docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> The data should still be there.

```bash
docker volume ls
docker volume inspect pg_data
```

> **Screenshot 5:** Take a screenshot showing the `SELECT` result after
> container recreation, and the `docker volume inspect` output.
>
> `[insert screenshot]`
> <img width="464" height="180" alt="Screenshot 5" src="https://github.com/user-attachments/assets/bf68ba66-736a-474c-8755-32b1bb72a7c4" />


### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker volume rm pg_data
```

### Questions for Section 4

**Question 4.1:** `docker volume inspect pg_data` shows a `Mountpoint` on
the host filesystem. Why is it still recommended to use named volumes instead
of bind-mounting that path directly with `-v /var/lib/docker/volumes/...`?

> *Your answer:*Named volumes are managed by Docker, while bind‑mounting the internal path bypasses Docker’s safety and stability guarantees.
Using the host path directly (-v /var/lib/docker/volumes/...) is risky because:
Docker may change that internal directory structure, breaking your setup.
Permissions and SELinux/AppArmor rules may not match what PostgreSQL expects.
You can accidentally delete or corrupt the volume because you’re touching Docker’s private storage area.Named volumes give you:
A stable, portable, Docker‑managed location
Safe lifecycle management (docker volume rm, backups, migration)
Isolation from host‑level misconfiguration.

**Question 4.2:** You want to back up the database. Which `docker` command
lets you copy files out of a running container, and how would you copy the
volume contents to a `.tar.gz` archive on the host?

> *Your answer:*You use docker cp to copy files out of (or into) a running container.
To back up the volume, you can archive the database directory inside the container and copy 

---

## 5 – Two Containers and the Network Problem

### Step 1 – Reproduce the Connectivity Failure

Start PostgreSQL:

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Start a second container and try to reach the first via `localhost`:

```bash
docker run --rm -it postgres:16 \
    psql -h localhost -U postgres -c "SELECT 1;"
```

> This should fail with a connection refused error.

> **Screenshot 6:** Take a screenshot showing the connection error.
>
> `[insert screenshot]`
> <img width="447" height="99" alt="Screenshot 6" src="https://github.com/user-attachments/assets/69583c16-7961-46a1-bfd8-ccaa3082b1e2" />


### Step 2 – Fix It With a Custom Bridge Network

```bash
docker network create mein-netz

docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    --network mein-netz \
    -d postgres:16

docker run --rm -it \
    --network mein-netz \
    postgres:16 \
    psql -h pg -U postgres -c "SELECT 1;"
```

> Notice that `-h pg` uses the **container name** as the hostname — Docker's
> internal DNS resolves it automatically.

```bash
docker network inspect mein-netz
```

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker network rm mein-netz
docker volume rm pg_data
```

### Questions for Section 5

**Question 5.1:** Without a custom bridge network, containers are placed on
the default bridge. Why can containers on the default bridge **not** resolve
each other by name, while containers on a user-defined bridge can?

> *Your answer:*Containers on the default bridge cannot resolve each other by name because Docker does not run an embedded DNS server for that network. Name‑to‑IP resolution simply isn’t provided there.

On a user‑defined bridge, Docker automatically enables an internal DNS service. Every container is registered in that DNS, so containers can resolve each other by name (e.g., postgres, backend, frontend) without needing hard‑coded IPs.

**Question 5.2:** You could find the IP address of the `pg` container with
`docker inspect` and hard-code it. Why is using the container name as a
hostname strongly preferable?

> *Your answer:*Using the container name as a hostname is strongly preferable because Docker does not guarantee stable IP addresses, but it does guarantee stable, resolvable hostnames for containers on the same user‑defined network

---

## 6 – Docker Compose

Managing multiple `docker run` commands by hand is error-prone. Docker Compose
describes an entire multi-service application in a single YAML file.

### Step 1 – Create the Project

```bash
mkdir ~/dbms09_compose
cd ~/dbms09_compose
git init
git remote add origin git@github.com:<your-username>/dbms09_compose.git
```

### Step 2 – Write docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend

  api:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - ./api:/app
    command: >
      bash -c "pip install fastapi uvicorn psycopg2-binary --quiet
               && uvicorn main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    networks:
      - backend

volumes:
  pg_data:

networks:
  backend:
```

### Step 3 – Create the API Code

```bash
mkdir api
vim api/main.py
```

```python
import psycopg2
import psycopg2.extras
from fastapi import FastAPI

app = FastAPI(title="Studenten-API")

DB_CONFIG = {
    "dbname": "vorlesung",
    "user": "vorlesung",
    "password": "geheim",
    "host": "postgres",   # container name as hostname
    "port": 5432,
}

@app.get("/")
def root():
    return {"status": "API läuft"}

@app.get("/studenten")
def alle_studenten():
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute("SELECT id, matrikel, nachname, vorname FROM student ORDER BY nachname")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return rows
```

### Step 4 – Start and Test

```bash
docker compose up -d
docker compose ps
docker compose logs api
```

Wait for the API to start, then test:

```bash
curl http://localhost:8000/
curl http://localhost:8000/studenten
```

> The `/studenten` endpoint will return an empty list for now — that is
> expected. You will add the schema in Section 7.

> **Screenshot 7:** Take a screenshot showing `docker compose ps` and the
> `curl /` response.
>
> `[insert screenshot]`
<img width="608" height="49" alt="Screenshot 7a" src="https://github.com/user-attachments/assets/842f41c5-d82f-48d3-82d4-5b8afe92391d" /><img width="463" height="32" alt="Screenshot 7b" src="https://github.com/user-attachments/assets/ad705d7b-9a0f-44a9-8df1-f91cc3248e34" />


### Step 5 – Observe Compose Networking

```bash
docker network ls
docker network inspect dbms09_compose_backend
```

> Compose automatically prefixes network names with the project directory name.

### Step 6 – Take It Down

```bash
docker compose down
docker compose down -v   # also removes the named volume
```

> **Question:** What is the difference between `down` and `down -v`?
> When would you use each?

### Step 7 – Commit

```bash
git add docker-compose.yml api/main.py
git commit -m "feat: initial docker-compose setup with postgres and api"
git push -u origin main
```

### Questions for Section 6

**Question 6.1:** `depends_on: postgres` ensures the `postgres` service
starts before `api`. Does it guarantee that PostgreSQL is **ready to accept
connections** when the API starts? What is the correct way to handle this?

> *Your answer:*No, depends_on: postgres does not guarantee that PostgreSQL is ready.
It only guarantees that the Postgres container starts first, but the database might still be loading and not ready for connections.Use a healthcheck so the API waits until PostgreSQL is healthy.

**Question 6.2:** The `api` service uses `volumes: - ./api:/app` (a bind
mount). What is the advantage of this during development compared to
`COPY`-ing the code into an image at build time? 


> *Your answer:*Using a bind mount (./api:/app) means the code on your computer is directly visible inside the container.So when you change a file, the running container sees the change immediately — no rebuild needed.
If you used COPY in the image, you would have to rebuild the image every time you change your code, which is slow and annoying during development.

---

## 7 – Init Script for PostgreSQL

The official `postgres` image runs all scripts placed in
`/docker-entrypoint-initdb.d/` on first start. This lets you initialise the
schema automatically.

### Step 1 – Write init.sql

```bash
vim init.sql
```

```sql
CREATE TABLE IF NOT EXISTS student (
    id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    matrikel  CHAR(8)      NOT NULL UNIQUE,
    nachname  VARCHAR(100) NOT NULL,
    vorname   VARCHAR(100) NOT NULL,
    email     VARCHAR(200)
);

INSERT INTO student (matrikel, nachname, vorname, email) VALUES
    ('12345678', 'Meier',   'Anna',  'a.meier@stud.thga.de'),
    ('23456789', 'Schmidt', 'Ben',   'b.schmidt@stud.thga.de'),
    ('34567890', 'Yilmaz',  'Ceren', 'c.yilmaz@stud.thga.de'),
    ('45678901', 'Nguyen',  'David', 'd.nguyen@stud.thga.de');
```

### Step 2 – Mount the Script in docker-compose.yml

Add a bind mount to the `postgres` service so the script lands in the init
directory:

```yaml
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
```

### Step 3 – Reinitialise and Test

The init script only runs when the data directory is empty. Remove the
existing volume first:

```bash
docker compose down -v
docker compose up -d
```

Wait a moment, then query the data:

```bash
curl http://localhost:8000/studenten
```

> You should now see all four students in the JSON response.

> **Screenshot 8:** Take a screenshot showing the `curl /studenten` response
> with all four rows.
>
> `[insert screenshot]`
> <img width="619" height="55" alt="Screenshot 8" src="https://github.com/user-attachments/assets/9889e1ab-4981-49d9-b5cc-3a955876c478" />


### Step 4 – Commit

```bash
git add init.sql docker-compose.yml
git commit -m "feat: add init.sql for automatic schema and seed data"
git push
```

### Questions for Section 7

**Question 7.1:** You run `docker compose down` (without `-v`), change
`init.sql`, and run `docker compose up -d` again. The schema change does
**not** appear in the database. Why not, and how do you force re-initialisation?

> *Your answer:*The schema change does not appear because the init.sql only runs when the data directory is empty.
When you run docker compose down without -v, the volume is kept, so PostgreSQL sees an existing data directory and skips the init script.You must delete the volume so PostgreSQL starts with an empty data directory.

**Question 7.2:** `GENERATED ALWAYS AS IDENTITY` is used instead of
`SERIAL`. What is the practical difference? Which one is the modern
SQL-standard approach?

> *Your answer:*SERIAL is the old PostgreSQL way to make an auto‑incrementing ID.
It works, but it is not part of the official SQL standard and creates a hidden sequence.
GENERATED ALWAYS AS IDENTITY is the modern, SQL‑standard way.
PostgreSQL recommends using it now.
Practical difference (simple)
SERIAL
Old method
Creates a hidden sequence
Not fully controlled by SQL standard
Harder to manage in migrations

IDENTITY (GENERATED ALWAYS AS IDENTITY)
Modern method
Follows the SQL standard
Sequence is managed cleanly by PostgreSQL
Better for tools, migrations, and future compatibility

---

## 8 – Secrets via .env

Passwords must not appear in `docker-compose.yml` because that file is
committed to version control.

### Step 1 – Create .env

```bash
vim .env
```

```
POSTGRES_USER=vorlesung
POSTGRES_PASSWORD=geheim
POSTGRES_DB=vorlesung
```

### Step 2 – Reference Variables in docker-compose.yml

Replace the hard-coded values in the `postgres` service:

```yaml
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

Also update `api/main.py` to read the password from the environment:

```python
import os

DB_CONFIG = {
    "dbname": os.environ.get("POSTGRES_DB", "vorlesung"),
    "user": os.environ.get("POSTGRES_USER", "vorlesung"),
    "password": os.environ.get("POSTGRES_PASSWORD", ""),
    "host": "postgres",
    "port": 5432,
}
```

Add `env_file` to the `api` service in `docker-compose.yml`:

```yaml
  api:
    ...
    env_file: .env
```

### Step 3 – Gitignore .env

```bash
echo ".env" >> .gitignore
```

### Step 4 – Restart and Verify

```bash
docker compose down -v
docker compose up -d
curl http://localhost:8000/studenten
```

> The response should be identical to before.

### Step 5 – Commit

```bash
git add docker-compose.yml api/main.py .gitignore
git commit -m "feat: move credentials to .env file"
git push
```

> **Do not add `.env` to the commit.** Confirm with `git status` that it
> is untracked.

> **Screenshot 9:** Take a screenshot showing `git status` confirming
> `.env` is not staged, and the working `curl` response.
>
> `[insert screenshot]`
> <img width="292" height="94" alt="Screenshot 9" src="https://github.com/user-attachments/assets/bbbc7c70-2937-4d66-979d-8aff22456ca3" />


### Questions for Section 8

**Question 8.1:** A teammate clones your repository and runs
`docker compose up -d`. The application fails because `.env` is missing.
What is the standard practice to document which variables are required
without committing the actual secrets?

> *Your answer:*You should not commit the real .env file, because it contains secrets.
But your teammate still needs to know which variables to create.Create a file called .env.example (or .env.template) and commit that to Git.It contains only the variable names, without real secrets:DB_HOST=
DB_USER=
DB_PASSWORD=
API_KEY=


**Question 8.2:** Even with `.env` excluded from git, the password is still
stored in plain text on disk. Name one mechanism Docker provides for
production-grade secret management that avoids plain-text env files entirely.

> *Your answer:*Yes, .env is still plain text, so it’s not safe for production.
Docker provides Docker Secrets, which store passwords in a secure, encrypted way instead of in a normal file.
The secret is mounted into the container in a protected location, not as an environment variable.

---

## 9 – Multi-Stage Build

A Python image that includes `pip`, build tools, and cache is larger than
necessary. A Multi-Stage Build separates dependency installation from the
final runtime image.

### Step 1 – Create an api/Dockerfile

```bash
vim api/Dockerfile
```

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Add pyproject.toml

```bash
vim api/pyproject.toml
```

```toml
[project]
name = "studenten-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "psycopg2-binary",
]
```

### Step 3 – Switch the api Service to Use the Build

Replace the `image:` key with `build:` in the `api` service:

```yaml
  api:
    build: ./api
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    env_file: .env
    networks:
      - backend
```

Remove the `volumes: - ./api:/app` bind mount and the `command:` key — they
were only needed for the quick-start approach.

### Step 4 – Build and Test

```bash
docker compose down -v
docker compose build
docker images    # compare sizes
docker compose up -d
curl http://localhost:8000/studenten
```

> **Screenshot 10:** Take a screenshot showing `docker images` with the
> final image size and the working `curl` response.
>
> `[insert screenshot]`
> <img width="306" height="92" alt="Screenshot 10a" src="https://github.com/user-attachments/assets/37d3394c-68c7-41d3-b516-ab5b41b551ac" /><img width="443" height="29" alt="Screenshot 10b" src="https://github.com/user-attachments/assets/7a686a1a-9dc7-4734-92fb-3f1f85368603" />



### Step 5 – Commit

```bash
git add api/Dockerfile api/pyproject.toml docker-compose.yml
git commit -m "feat: multi-stage Dockerfile for slim production image"
git push
```

### Questions for Section 9

**Question 9.1:** `COPY --from=builder /app/.venv .venv` copies the virtual
environment from the builder stage. The final image does not contain `pip` or
`uv`. What security advantage does this provide?

> *Your answer:*Final image has no pip, no uv ,Attackers cannot install anything
Smaller, safer, harder to abuse
This is a common best practice:
Builder stage = tools
Final stage = only what is needed to run the app

**Question 9.2:** The builder stage installs dependencies from `pyproject.toml`
before copying the application code. Why does this ordering improve build
cache efficiency when you frequently change only `main.py`?

> *Your answer:*Dependencies installed first → cached
Changing your code doesn’t break the cache
Builds become much faster during development

---

## 10 – Non-Root User

Containers run as `root` by default. If an attacker escapes the container,
they have root on the host.

### Step 1 – Add a Non-Root User to the Dockerfile

Open `api/Dockerfile` and add the user before the `CMD`:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN adduser --disabled-password --gecos "" appuser
USER appuser
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Rebuild and Verify

```bash
docker compose build
docker compose up -d
docker compose exec api whoami
```

> Expected output: `appuser`

> **Screenshot 11:** Take a screenshot showing `docker compose exec api whoami`
> returning `appuser`.
>
> `[insert screenshot]`
> <img width="398" height="29" alt="Screenshot 11" src="https://github.com/user-attachments/assets/ce4d277d-60e1-4427-8142-0c0c89fa93f2" />


### Step 3 – Commit

```bash
git add api/Dockerfile
git commit -m "feat: run api container as non-root appuser"
git push
```

### Questions for Section 10

**Question 10.1:** The `USER appuser` instruction is placed after
`COPY . .`. Why would placing it *before* `COPY` cause a permission problem?

> *Your answer:*USER appuser before COPY → appuser cannot write → permission error
USER appuser after COPY → root copies files → no problem

**Question 10.2:** State the **Principle of Least Privilege** in one
sentence, and name one other place in a typical web application stack
(outside of containers) where this principle is applied.

> *Your answer:*Least privilege = minimal permissions
Example = restricted database user (cannot do dangerous operations)

---

## 11 – Reflection

**Question A – The Monolith Anti-Pattern:**  
Section 6 of the lecture shows a Dockerfile that runs both PostgreSQL and
FastAPI in a single container. Describe two concrete operational problems
this causes in a production environment.

> *Your answer:*No independent scaling → bad performance and wasted resources
No independent restarts/updates → downtime and fragile deployments

**Question B – Volume vs. Bind Mount:**  
Compare named volumes and bind mounts. When is each type appropriate?

> *Your answer:*Named volume = stable, safe, Docker‑managed → best for production data
Bind mount = live editing, instant updates → best for development

**Question C – Compose and Reproducibility:**  
A colleague says: "I can just write the `docker run` commands in a shell
script — why do I need `docker-compose.yml`?" Give two specific advantages
of Compose over a shell script of `docker run` commands.

> *Your answer:*Compose = one clean file → easy, reproducible setup
Compose = automatic networks + dependencies → less manual work

**Question D – The Complete Chain:**  
You have now built and containerised the full stack: PostgreSQL in a
container with a named volume and init script → FastAPI in a slim
non-root image → both orchestrated by Docker Compose with credentials
in `.env`. Describe in two sentences what each layer contributes to
**portability** and **security**.

> *Your answer:*PostgreSQL + named volume + init script:  
Portable database setup; secure storage managed by Docker.
FastAPI in slim non‑root image:  
Portable small image; safer because it doesn’t run as root.
Docker Compose + .env:  
Portable multi‑service configuration; secrets kept out of the image for better security.

---

## Further Reading

- [Docker – Get started](https://docs.docker.com/get-started/)
- [Docker – Volumes](https://docs.docker.com/storage/volumes/)
- [Docker – Networking overview](https://docs.docker.com/network/)
- [Docker Compose – Reference](https://docs.docker.com/compose/compose-file/)
- [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [postgres Docker Hub – Environment variables](https://hub.docker.com/_/postgres)
- Lecture 09 handout
