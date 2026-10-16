# Containerizing an Application with Docker

A concise guide to packaging an application and its dependencies into a
Docker image, then running it as a container.

## 1. Key Concepts

-   **Image:** Read-only package containing application code,
    dependencies, and filesystem.
-   **Container:** A running instance of an image.
-   **Dockerfile:** Instructions used to build an image.
-   **Build context:** Files available to Docker during a build.
-   **Registry:** A service for storing and distributing images, such as
    Docker Hub.

## 2. Prerequisites

-   The application runs locally.
-   Docker is installed and running.

Check the installation:

``` bash
docker --version
docker info
```

## 3. Prepare the Application

Before building:

-   List runtime dependencies in a file such as `requirements.txt`.
-   Pin dependency versions for repeatable builds.
-   Read configuration and secrets from environment variables.
-   Make the application listen on `0.0.0.0`, not only `127.0.0.1`.
-   Use a production server for deployment.

For Flask, the built-in server is suitable for development only. A
production deployment should use a server such as Gunicorn.

## 4. Create `.dockerignore`

Create a `.dockerignore` file in the project root to keep unnecessary
files and secrets out of the build context.

``` text
__pycache__/
*.py[cod]
.venv/
venv/
.git/
.env
*.log
.pytest_cache/
.idea/
.vscode/
```

Adjust this list for your project. For Node.js, also exclude
`node_modules/`.

## 5. Create a Dockerfile

For a Python application, create a file named `Dockerfile` in the
project root. This example assumes `requirements.txt` and `app.py` exist
and the app listens on port `5000`.

``` dockerfile
FROM python:3.13-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --create-home appuser
USER appuser

EXPOSE 5000
CMD ["python", "app.py"]
```

For production, replace the final command with the appropriate
production server command, for example:

``` dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

Install Gunicorn in `requirements.txt` if using this command. In
`app:app`, the first `app` is the Python module name and the second is
the application object.

### Important instructions

-   `FROM` selects the base image. Use a specific version rather than
    `latest`.
-   `WORKDIR` sets the working directory inside the container.
-   Copying the dependency file before the source code allows Docker to
    cache dependency installation.
-   `USER` runs the application without root privileges.
-   `EXPOSE` documents the container port; it does not publish it.
-   `CMD` defines the default start command.

The `python:3.13-slim` image is a practical general default. Confirm
that the Python version matches your application and dependencies.

## 6. Build the Image

Run these commands from the directory containing the Dockerfile:

``` bash
docker build -t my-app-image:1.0.0 .
docker images
```

The final `.` specifies the build context.

## 7. Run the Container

``` bash
docker run --name my-app -p 5000:5000 my-app-image:1.0.0
```

Open `http://localhost:5000` to test the application. The `-p` option
maps a host port to a container port.

To run in the background:

``` bash
docker run -d \
  --name my-app \
  -p 5000:5000 \
  --restart unless-stopped \
  my-app-image:1.0.0
```

Pass configuration at runtime with `-e KEY=value` or `--env-file .env`.
Keep `.env` out of version control and the Docker image.

Use Docker volumes for data that must persist after a container is
removed.

## 8. Manage and Troubleshoot Containers

``` bash
docker ps
docker ps -a
docker logs my-app
docker logs -f my-app
docker exec -it my-app sh
docker stop my-app
docker start my-app
docker rm my-app
docker rmi my-app-image:1.0.0
```

If a container exits or the app is unreachable, check its logs first.
Also verify the start command, installed dependencies, port mapping, and
that the app listens on `0.0.0.0`.

Common issues:

  -----------------------------------------------------------------------
  Problem                             Likely fix
  ----------------------------------- -----------------------------------
  Browser cannot connect              Check the bind address and `-p`
                                      mapping.

  Port already allocated              Choose another host port or stop
                                      the process using it.

  `ModuleNotFoundError`               Add the missing dependency to
                                      `requirements.txt`.

  `COPY` fails                        Check the build context and
                                      `.dockerignore`.

  Permission denied                   Give the non-root user access to
                                      required files or directories.

  Alpine package build fails          Try a Debian-based `-slim` image.
  -----------------------------------------------------------------------

## 9. Docker Compose (Optional)

Use Compose when an application depends on services such as a database.
Example:

``` yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Set `POSTGRES_PASSWORD` in the environment or `.env` file; do not commit
real credentials. Within Compose, connect to the database using the
service name `db`, not `localhost`.

Start and stop the stack:

``` bash
docker compose up --build
docker compose down
```

## 10. Best-Practice Checklist

-   [ ] The application works locally before containerization.
-   [ ] Dependencies and base image versions are pinned appropriately.
-   [ ] `.dockerignore` excludes unnecessary files and secrets.
-   [ ] Configuration is supplied at runtime, not baked into the image.
-   [ ] The application listens on `0.0.0.0`.
-   [ ] The container runs as a non-root user.
-   [ ] A production-grade server is used for production.
-   [ ] Logs are written to standard output and standard error.
-   [ ] Persistent data is stored in volumes where needed.
-   [ ] The image is tested and scanned for vulnerabilities.
