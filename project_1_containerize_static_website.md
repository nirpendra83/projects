# Project 1: Containerize a Static Website with Docker

> **Goal:** Build and run a simple HTML/CSS/JavaScript website in a
> Docker container using Nginx. Then use a bind mount so that local file
> changes appear without rebuilding the image.

This is a beginner-friendly first project for learning Docker
fundamentals.

## What you will build

A small static website served by Nginx in a container and accessible at:

-   Website: `http://localhost:8080`

You can use plain HTML and CSS, with optional JavaScript. No backend or
database is required.

## Skills you will learn

-   Writing a basic `Dockerfile`.
-   Understanding `FROM`, `COPY`, `WORKDIR`, `EXPOSE`, and `CMD`.
-   Building an image and running a container.
-   Mapping a host port to a container port.
-   Using bind mounts to share files between your machine and a
    container.
-   Understanding the difference between rebuilding an image and
    changing mounted files.

## Prerequisites

Install Docker Desktop (Windows/macOS) or Docker Engine (Linux). Verify
that Docker is running:

``` bash
docker --version
docker info
```

If `docker info` fails, start Docker Desktop or the Docker service
before continuing.

## Step 1: Create the project

Create this directory structure:

``` text
static-site/
├── index.html
├── style.css
└── Dockerfile
```

### `index.html`

``` html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Docker Static Website</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <main class="card">
    <h1>Hello, Docker!</h1>
    <p>This static website is served by Nginx in a container.</p>
    <p>Edit <code>index.html</code> or <code>style.css</code>, then refresh the browser.</p>
  </main>
</body>
</html>
```

### `style.css`

``` css
body {
  margin: 0;
  min-height: 100vh;
  display: grid;
  place-items: center;
  font-family: Arial, sans-serif;
  background: #f1f5f9;
  color: #172033;
}

.card {
  max-width: 560px;
  margin: 24px;
  padding: 32px;
  border-radius: 16px;
  background: white;
  box-shadow: 0 8px 30px rgb(0 0 0 / 10%);
}

h1 {
  color: #166534;
}
```

## Step 2: Write the Nginx Dockerfile

Create a file named exactly `Dockerfile` (without a file extension):

``` dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
COPY style.css /usr/share/nginx/html/style.css

EXPOSE 80
```

### What each instruction does

-   `FROM nginx:alpine`: starts from the lightweight Nginx image.
-   `COPY`: copies website files into Nginx's default document
    directory.
-   `EXPOSE 80`: documents the port the containerized service listens
    on. It does **not** publish the port to your computer by itself.
-   No custom `CMD` is required: the official Nginx image already
    defines a command that starts Nginx in the foreground.

## Step 3: Build the image

Run these commands from inside the `static-site` directory:

``` bash
docker build -t my-static-site:1.0 .
docker image ls
```

The `.` at the end means that Docker uses the current directory as the
build context.

## Step 4: Run the container

``` bash
docker run --name static-site -d -p 8080:80 my-static-site:1.0
docker ps
```

Open `http://localhost:8080` in your browser.

### Understand the port mapping

``` text
Your computer:8080  --->  Container:80  --->  Nginx
```

The left side (`8080`) is the host port. The right side (`80`) is the
port inside the container.

If port `8080` is already in use, choose another host port, for example
`-p 8081:80`, and browse to `http://localhost:8081`.

## Step 5: Update the website

Change the heading in `index.html`, save the file, and refresh the
browser.

With the image-only setup above, the file inside the running container
does not change when you edit the host file. You must rebuild and
recreate the container to see the new version:

``` bash
docker build -t my-static-site:1.1 .
docker rm -f static-site
docker run --name static-site -d -p 8080:80 my-static-site:1.1
```

This demonstrates the difference between files copied into an image at
build time and files shared using a bind mount.

## Step 6: Enable quick local updates with a bind mount

For local development, stop and remove the original container:

``` bash
docker rm -f static-site
```

Run Nginx with the project directory mounted into its document root.

### Linux / macOS / Git Bash

Run this command from the project directory:

``` bash
docker run --name static-site -d \
  -p 8080:80 \
  --mount "type=bind,source=$(pwd),target=/usr/share/nginx/html,readonly" \
  nginx:alpine
```

### Windows PowerShell

Run this command from the project directory:

``` powershell
docker run --name static-site -d `
  -p 8080:80 `
  --mount "type=bind,source=$($PWD.Path),target=/usr/share/nginx/html,readonly" `
  nginx:alpine
```

Now edit `index.html` or `style.css`, save, and refresh the browser. The
updated files are available in the container through the bind mount, so
you do not need to rebuild the image for each content change.

**Important:** This is refresh-to-see-changes, not automatic browser
live reload. Nginx does not inject live-reload behavior. The mount is
read-only inside the container to prevent the web server from modifying
your source files.

The `--mount` source path must resolve to your actual project directory.
If you use a different terminal, adjust the path syntax accordingly.

## Optional: Use Node.js `http-server`

This alternative uses Node.js and `http-server` instead of Nginx. It is
useful for demonstrating a different static file server, but
`http-server` does not provide automatic browser live reload by itself.

Create `Dockerfile.http-server`:

``` dockerfile
FROM node:alpine

RUN npm install --global http-server
WORKDIR /app
EXPOSE 8080
CMD ["http-server", ".", "-p", "8080", "-c-1"]
```

Build and run it from the project directory:

``` bash
docker build -f Dockerfile.http-server -t my-static-site-node:1.0 .
docker run --name static-site-node -d \
  -p 8080:8080 \
  --mount "type=bind,source=$(pwd),target=/app,readonly" \
  my-static-site-node:1.0
```

On Windows PowerShell, use this bind-mount form instead:

``` powershell
docker run --name static-site-node -d `
  -p 8080:8080 `
  --mount "type=bind,source=$($PWD.Path),target=/app,readonly" `
  my-static-site-node:1.0
```

Browse to `http://localhost:8080`. The `-c-1` option disables
`http-server`'s caching, which helps when testing changes. It does not
automatically refresh the browser.

To use the Node alternative, stop the Nginx container first if it is
still using host port `8080`.

## Troubleshooting

### The website does not open

``` bash
docker ps
docker logs static-site
docker port static-site
```

Check that the container is running and that the host port mapping is
correct.

### Port 8080 is already in use

Use a different host port:

``` bash
docker run --name static-site -d -p 8081:80 nginx:alpine
```

Then browse to `http://localhost:8081`. For this quick example, the
default Nginx image serves its welcome page unless you mount or copy
your website files.

### Changes do not appear

-   Confirm that you started the container with the bind mount.
-   Check that you edited the files in the directory mounted into the
    container.
-   Refresh the browser; try a hard refresh if cached content remains.
-   Check the mount path and container logs.

### Remove the lab container

``` bash
docker rm -f static-site
```

If you used the Node alternative:

``` bash
docker rm -f static-site-node
```

## Challenges for students

1.  Add an `about.html` page and link to it from `index.html`.
2.  Add a logo or image and make sure it is served correctly.
3.  Change the host port from `8080` to `8081`.
4.  Compare the copied-file workflow with the bind-mount workflow.
5.  Inspect the container with `docker exec -it static-site sh`.
6.  Explain why `EXPOSE 80` does not replace `-p 8080:80`.
7.  Explain the difference between an image, a container, and a bind
    mount.
8.  Optional: Configure a separate development server that provides
    browser live reload.

## What success looks like

-   The site loads at `http://localhost:8080`.
-   The student can explain the Dockerfile instructions.
-   The student can explain host-to-container port mapping.
-   With a bind mount, edits to local HTML/CSS files become visible
    after refreshing the browser, without rebuilding the image.
-   The student can stop and remove the container cleanly.

## Why this project matters

This small project teaches the foundation of containerized applications:
packaging, ports, filesystem boundaries, and repeatable execution. Those
concepts are used again in later projects involving APIs, CI/CD
pipelines, Kubernetes, and production deployments.
