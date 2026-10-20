# Docker Learning Path: Projects 2--10

This guide continues from **Project 1: Containerize a Static Website**.
These projects progress from frontend packaging to multi-container apps,
storage, reverse proxies, and CI/CD.

> **Safety:** These are learning labs, not production-ready deployments.
> Keep lab services bound to localhost, use demo-only credentials, and
> do not expose development tools, databases, or insecure dashboards to
> the internet.

## Project 2: Build a Developer Portfolio with a Multi-Stage Docker Build

### Goal and skills

Build a React/Vite portfolio and serve its compiled files with Nginx.
Learn multi-stage builds, `.dockerignore`, and the difference between
development and production.

### Create the app

``` bash
npm create vite@latest my-portfolio -- --template react
cd my-portfolio
npm install
npm run dev
```

### `.dockerignore`

``` text
node_modules
dist
.git
.gitignore
Dockerfile*
compose*.yaml
npm-debug.log*
.env
.env.*
```

### `Dockerfile`

``` dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

`npm ci` expects a lockfile such as `package-lock.json`. If the project
has none, run `npm install` once to create it. The final image contains
Nginx and the built static files, not Node.js dependencies.

### Build and run

``` bash
docker build -t my-portfolio:1.0 .
docker run --rm --name my-portfolio -p 8080:80 my-portfolio:1.0
```

Open `http://localhost:8080`.

### Optional development mode with Vite hot reload

Create `compose.dev.yaml`:

``` yaml
services:
  frontend:
    image: node:22-alpine
    working_dir: /app
    ports:
      - "127.0.0.1:5173:5173"
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    command: sh -c "npm ci && npm run dev -- --host 0.0.0.0"
volumes:
  node_modules:
```

Run `docker compose -f compose.dev.yaml up`, then open
`http://localhost:5173`. Vite provides browser hot reload in
development; this is separate from the production Nginx image.

**Challenge:** Compare the production image size with the development
environment and explain why `node_modules` is not needed at runtime.

------------------------------------------------------------------------

## Project 3: Run a PHP and MySQL Application with Docker Compose

### Goal

Learn multi-container networking, service-name DNS, environment
variables, health checks, and persistent database storage.

### Folder structure

``` text
lamp-lab/
├── compose.yaml
├── Dockerfile
└── web/
    └── index.php
```

### `Dockerfile`

``` dockerfile
FROM php:8.3-apache
RUN docker-php-ext-install mysqli
COPY web/ /var/www/html/
```

### `web/index.php`

``` php
<?php
$host = getenv('DB_HOST') ?: 'db';
$user = getenv('DB_USER') ?: 'appuser';
$password = getenv('DB_PASSWORD') ?: 'localdevpassword';
$database = getenv('DB_NAME') ?: 'appdb';

$mysqli = new mysqli($host, $user, $password, $database);

if ($mysqli->connect_error) {
    http_response_code(503);
    echo "Database connection is not ready.";
    exit;
}

echo "Connected to MySQL!";
$mysqli->close();
```

This is a connectivity demonstration, not a complete production
application.

### `compose.yaml`

``` yaml
services:
  web:
    build: .
    ports:
      - "127.0.0.1:8081:80"
    environment:
      DB_HOST: db
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: localdevpassword
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: localdevpassword
      MYSQL_ROOT_PASSWORD: localrootpassword
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -uroot -p\"$$MYSQL_ROOT_PASSWORD\" --silent"]
      interval: 5s
      timeout: 5s
      retries: 20

volumes:
  db_data:
```

The passwords are local-lab examples only. Use secrets management and
unique credentials in shared or production environments.

### Run

``` bash
docker compose up --build -d
docker compose ps
docker compose logs -f web
```

Open `http://localhost:8081`. The PHP container reaches MySQL using the
hostname `db`, which is the Compose service name.

### Cleanup

``` bash
docker compose down
```

To also permanently delete the lab database volume:

``` bash
docker compose down -v
```

**Challenge:** Add a form that inserts a record using prepared
statements, then recreate the containers and verify that the data
persists.

------------------------------------------------------------------------

## Project 4: Run WordPress with a Database

### Goal

Deploy WordPress and MariaDB, learn persistent volumes, and understand
how an application depends on a database.

Create `compose.yaml`:

``` yaml
services:
  wordpress:
    image: wordpress:latest
    ports:
      - "127.0.0.1:8082:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: localwpdevpassword
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: mariadb:11
    environment:
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wpuser
      MARIADB_PASSWORD: localwpdevpassword
      MARIADB_ROOT_PASSWORD: localrootpassword
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 5s
      timeout: 5s
      retries: 20
    restart: unless-stopped

volumes:
  wp_data:
  db_data:
```

These credentials are for a disposable local lab only. For repeatable
deployments, pin tested image versions instead of relying blindly on
`latest`.

Start the stack:

``` bash
docker compose up -d
docker compose ps
docker compose logs -f wordpress
```

Open `http://localhost:8082` and complete the WordPress setup wizard.

The `wp_data` volume stores WordPress files; `db_data` stores database
files. For theme development, mount a dedicated theme directory rather
than replacing the whole WordPress document root.

**Challenges:** Install a theme, restart the containers, and verify the
site persists. Then practise a database backup and restore.

Cleanup:

``` bash
docker compose down
```

Use `docker compose down -v` only when you intend to delete all lab
data.

------------------------------------------------------------------------

## Project 5: Run a Browser-Based IDE with code-server

### Goal

Run a browser-based code editor and persist files in a host directory.

**Security:** An IDE exposes a terminal and files. Keep it local for
this lab. Do not mount SSH keys or cloud credentials into an untrusted
container, and do not expose the service publicly with a weak password.

Create a `projects/` directory and set a strong local password in your
terminal.

Linux/macOS/Git Bash:

``` bash
export CODE_SERVER_PASSWORD='replace-with-a-strong-local-password'
```

PowerShell:

``` powershell
$env:CODE_SERVER_PASSWORD = "replace-with-a-strong-local-password"
```

Create `compose.yaml`:

``` yaml
services:
  code-server:
    image: lscr.io/linuxserver/code-server:latest
    container_name: code-server-lab
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: "Asia/Kolkata"
      PASSWORD: "${CODE_SERVER_PASSWORD:?Set CODE_SERVER_PASSWORD first}"
    ports:
      - "127.0.0.1:8443:8443"
    volumes:
      - ./projects:/config/workspace
    restart: unless-stopped
```

Check the current image documentation for supported environment
variables and paths, and pin a tested version for repeatable use.

Run:

``` bash
docker compose up -d
docker compose logs -f code-server
```

Open `http://localhost:8443` and sign in with the password you set.

**Challenges:** Create a file in the browser editor, confirm it appears
under `projects/` on your host, and verify it remains after restarting
the container.

------------------------------------------------------------------------

## Project 6: Build a Simple File-Drop API with Flask

### Goal

Learn application containerization, file uploads, bind mounts, and basic
file-handling safeguards.

Folder structure:

``` text
file-drop/
├── app.py
├── requirements.txt
└── Dockerfile
```

### `requirements.txt`

``` text
Flask>=3.0,<4.0
Werkzeug>=3.0,<4.0
```

### `app.py`

``` python
import os
from pathlib import Path
from flask import Flask, request, jsonify
from werkzeug.utils import secure_filename

app = Flask(__name__)
UPLOAD_FOLDER = Path(os.environ.get("UPLOAD_FOLDER", "/uploads"))
UPLOAD_FOLDER.mkdir(parents=True, exist_ok=True)
app.config["MAX_CONTENT_LENGTH"] = 10 * 1024 * 1024  # 10 MiB

@app.get("/")
def index():
    return """
    <h1>File Drop Lab</h1>
    <form action="/upload" method="post" enctype="multipart/form-data">
      <input type="file" name="file" required>
      <button type="submit">Upload</button>
    </form>
    """

@app.post("/upload")
def upload():
    uploaded = request.files.get("file")
    if not uploaded or not uploaded.filename:
        return jsonify(error="Choose a file"), 400

    filename = secure_filename(uploaded.filename)
    if not filename:
        return jsonify(error="Invalid filename"), 400

    uploaded.save(UPLOAD_FOLDER / filename)
    return jsonify(message="Uploaded", filename=filename), 201

@app.errorhandler(413)
def too_large(_error):
    return jsonify(error="File too large; limit is 10 MiB"), 413

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

This example still lacks authentication, malware scanning, quotas, safe
download handling, and collision protection. Do not use it for sensitive
or publicly accessible uploads without adding those controls.

### `Dockerfile`

``` dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UPLOAD_FOLDER=/uploads
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN useradd --create-home --uid 10001 appuser \
    && mkdir -p /uploads \
    && chown -R appuser:appuser /app /uploads
USER 10001
EXPOSE 5000
CMD ["python", "app.py"]
```

Build and run from the project directory:

``` bash
docker build -t file-drop:1.0 .
mkdir -p uploads
docker run --rm --name file-drop \
  -p 127.0.0.1:5000:5000 \
  --mount "type=bind,source=$(pwd)/uploads,target=/uploads" \
  file-drop:1.0
```

For PowerShell, replace the mount with:

``` powershell
docker run --rm --name file-drop `
  -p 127.0.0.1:5000:5000 `
  --mount "type=bind,source=$($PWD.Path)\uploads,target=/uploads" `
  file-drop:1.0
```

Open `http://localhost:5000`. Test with curl:

``` bash
curl -F 'file=@sample.txt' http://localhost:5000/upload
```

**Challenges:** Add authentication, a file-extension policy, safe
duplicate-name handling, and tests for the 10 MiB limit.

------------------------------------------------------------------------

## Project 7: Run a Privacy-Friendly Analytics Platform

### Goal

Deploy Plausible or Umami and learn multi-service configuration,
persistent data, and reverse proxy concepts.

Analytics products often require specific combinations of databases,
secrets, environment variables, and application versions. A short
generic Compose file can look valid but fail at startup. For this
project:

1.  Choose Plausible or Umami.
2.  Follow the product's current official self-hosting documentation and
    Compose configuration.
3.  Generate secrets as instructed; do not use placeholder values such
    as `super_secret` in a real deployment.
4.  Keep the service bound to localhost while learning.
5.  Validate and start the stack:

``` bash
docker compose config
docker compose up -d
docker compose ps
docker compose logs -f
```

6.  Open the URL specified by the configuration, configure a test
    website, and add the tracking snippet provided by the platform to
    Project 1.

**Skills:** Multi-container applications, application configuration,
secrets, database persistence, reverse proxying, and privacy-aware
analytics.

**Challenge:** Back up the analytics database, restore it into a test
environment, and verify that the historical data remains available.

------------------------------------------------------------------------

## Project 8: Create a CI/CD Pipeline with Jenkins

### Goal

Run Jenkins locally, create a pipeline, connect a Git repository, and
automate a test or build.

Create `compose.yaml`:

``` yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-lab
    ports:
      - "127.0.0.1:8085:8080"
    volumes:
      - jenkins_home:/var/jenkins_home
    restart: unless-stopped

volumes:
  jenkins_home:
```

This intentionally does not mount the host Docker socket. Building
Docker images from Jenkins requires a deliberate builder design; the
host socket gives Jenkins powerful control over the host and should not
be mounted casually.

Start Jenkins:

``` bash
docker compose up -d
docker compose logs -f jenkins
docker exec jenkins-lab cat /var/jenkins_home/secrets/initialAdminPassword
```

Open `http://localhost:8085`, complete the setup wizard, and install the
suggested plugins.

Create a Pipeline job with this starting `Jenkinsfile`:

``` groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                echo 'Connect this job to your Git repository.'
            }
        }
        stage('Test') {
            steps {
                echo 'Replace this with the project test command.'
            }
        }
    }
}
```

Connect the job to a repository and replace the placeholder with real
test/build commands. Store credentials in Jenkins Credentials, never
directly in the Jenkinsfile.

**Challenges:** Build and test Project 2 from Jenkins, then add a
controlled deployment stage after securing credentials and permissions.

------------------------------------------------------------------------

## Project 9: Route Containers Through a Reverse Proxy with Traefik

### Goal

Learn hostname-based routing and service discovery for multiple
containers.

**Security note:** Access to the Docker socket grants significant
control over the Docker host. A read-only socket mount is not a complete
security boundary. Keep this lab local, avoid enabling an insecure
dashboard, and evaluate a restricted socket proxy or another provider
for production.

Create `compose.yaml`:

``` yaml
services:
  traefik:
    image: traefik:v3
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

  web:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`myapp.localhost`)"
      - "traefik.http.routers.web.entrypoints=web"
      - "traefik.http.services.web.loadbalancer.server.port=80"
    restart: unless-stopped
```

Start and inspect:

``` bash
docker compose up -d
docker compose ps
docker compose logs -f traefik
```

Open `http://myapp.localhost:8080`. If hostname resolution does not work
in your environment, test with:

``` bash
curl -H "Host: myapp.localhost" http://127.0.0.1:8080
```

**Challenges:** Add a second web container under another hostname. For a
public deployment, configure HTTPS and authentication using current
documentation.

------------------------------------------------------------------------

## Project 10: Build a Mini SaaS-Style Stack with Node.js, MongoDB, and Nginx

### Goal

Combine an API, database, reverse proxy, health checks, and persistent
storage.

### Folder structure

``` text
mini-saas/
├── compose.yaml
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   └── server.js
└── nginx/
    └── default.conf
```

### `backend/package.json`

``` json
{
  "name": "mini-saas-api",
  "version": "1.0.0",
  "private": true,
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "mongoose": "^8.0.0"
  }
}
```

Generate and commit `package-lock.json` by running `npm install` in
`backend/` before building.

### `backend/server.js`

``` javascript
const express = require("express");
const mongoose = require("mongoose");

const app = express();
app.use(express.json());

const port = Number(process.env.PORT || 5000);
const mongoUrl = process.env.MONGO_URL || "mongodb://mongo:27017/saas";

app.get("/", (_req, res) => {
  res.send("Mini SaaS API is running");
});

app.get("/health", (_req, res) => {
  const ready = mongoose.connection.readyState === 1;
  res.status(ready ? 200 : 503).json({
    status: ready ? "ok" : "database-not-ready"
  });
});

async function start() {
  await mongoose.connect(mongoUrl);
  app.listen(port, "0.0.0.0", () => {
    console.log(`API listening on port ${port}`);
  });
}

start().catch((error) => {
  console.error("Failed to start API:", error.message);
  process.exit(1);
});
```

This is a minimal connectivity example, not a complete SaaS application.
Add structured logging, graceful shutdown, validation, authentication,
and robust readiness/retry behavior before treating it as production
software.

### `backend/Dockerfile`

``` dockerfile
FROM node:22-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY server.js .
USER node
EXPOSE 5000
CMD ["node", "server.js"]
```

### `nginx/default.conf`

``` nginx
server {
    listen 80;

    location / {
        proxy_pass http://backend:5000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### `compose.yaml`

``` yaml
services:
  backend:
    build: ./backend
    environment:
      PORT: "5000"
      MONGO_URL: mongodb://mongo:27017/saas
    depends_on:
      mongo:
        condition: service_healthy
    restart: unless-stopped

  mongo:
    image: mongo:8
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 5s
      timeout: 5s
      retries: 20
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8089:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  mongo_data:
```

MongoDB has no authentication in this isolated local lab and its port is
not published to the host. Do not use this configuration for an
internet-facing or shared environment.

Run:

``` bash
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs -f backend
```

Open `http://localhost:8089` and `http://localhost:8089/health`.

**Challenges:** Add a CRUD API and data model, validate inputs, add
MongoDB authentication using secrets, run tests in CI, and add TLS if
deploying beyond localhost.

Cleanup:

``` bash
docker compose down
```

To delete the MongoDB lab data too:

``` bash
docker compose down -v
```

------------------------------------------------------------------------

## Recommended Teaching Order

  ------------------------------------------------------------------------
  Project                 Main concept            Level
  ----------------------- ----------------------- ------------------------
  1\. Static website      Dockerfile, ports, bind Beginner
                          mounts                  

  2\. Portfolio           Multi-stage builds and  Beginner
                          frontend build          

  3\. PHP + MySQL         Compose, networking,    Beginner--Intermediate
                          database volume         

  4\. WordPress           Stateful application    Beginner--Intermediate
                          and database            

  5\. Browser IDE         Persistent development  Intermediate
                          environment and         
                          security                

  6\. File-drop API       Python app, uploads,    Intermediate
                          filesystem permissions  

  7\. Analytics           Multi-service           Intermediate
                          configuration           

  8\. Jenkins             CI/CD pipeline and      Intermediate
                          credentials             

  9\. Reverse proxy       Routing and service     Intermediate
                          discovery               

  10\. Mini SaaS          API, database, proxy,   Intermediate
                          health checks           
  ------------------------------------------------------------------------

## General Docker Commands to Practise

``` bash
docker ps
docker ps -a
docker images
docker logs <container>
docker exec -it <container> sh
docker inspect <container>
docker stop <container>
docker rm <container>
docker compose ps
docker compose logs -f
docker compose down
```

## Final Learning Outcome

These projects take learners from a single static container to
multi-container applications, persistent storage, reverse proxying, and
CI/CD. The next step is to automate tests and image builds in GitLab
CI/CD, publish versioned images to a registry, and deploy an application
to Kubernetes or OpenShift.
