# Setting Up a CI Pipeline with GitHub Actions

A CI pipeline automatically tests code changes. This workflow runs tests
on pushes and pull requests to `main`, then builds and publishes a
Docker image only after tests pass on `main`.

## 1. Prerequisites

-   The application runs locally.
-   Dependencies are listed in `requirements.txt`.
-   Tests pass locally.
-   A working `Dockerfile` exists.
-   The project is hosted on GitHub, and a Docker Hub repository is
    available.

Check locally:

``` bash
pip install -r requirements.txt
pytest
```

## 2. Add Docker Hub Secrets

Create a Docker Hub access token with read/write permission. In GitHub,
open **Settings → Secrets and variables → Actions** and add:

-   `DOCKERHUB_USERNAME`: your Docker Hub username.
-   `DOCKERHUB_TOKEN`: your Docker Hub access token.

Never put credentials directly in the workflow file.

## 3. Create the Workflow

Create `.github/workflows/ci.yml` in the repository:

``` yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest

  build-and-push:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app-image:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app-image:${{ github.sha }}
```

Replace `my-app-image` with your Docker Hub repository name. Ensure the
Python version matches your application and Dockerfile.

## 4. How It Works

1.  **Test job:** Checks out the code, installs Python and dependencies,
    then runs `pytest`.
2.  **Build-and-push job:** Waits for tests to pass and runs only for
    pushes to `main`.
3.  **Secrets:** Provide Docker Hub credentials securely.
4.  **Image tags:** `latest` points to the newest published image; the
    commit SHA identifies the exact source revision.

Pull requests run tests but do not publish images.

## 5. Run and Verify

Commit and push the workflow:

``` bash
git add .github/workflows/ci.yml
git commit -m "Add CI pipeline"
git push origin main
```

In GitHub, open the **Actions** tab and inspect the workflow run. A
successful run should show green checks and the image tags in Docker
Hub.

Test the published image:

``` bash
docker pull <dockerhub-username>/my-app-image:latest
docker run -p 5000:5000 <dockerhub-username>/my-app-image:latest
```

## 6. Common Problems

  -----------------------------------------------------------------------
  Problem                             Check
  ----------------------------------- -----------------------------------
  Workflow does not start             File location, YAML syntax, and
                                      branch triggers.

  Tests fail only in CI               Dependencies, Python version,
                                      environment variables, or
                                      local-only files.

  Docker Hub login/push fails         Secret names, token permissions,
                                      and repository name.

  Publish job is skipped              Tests failed, or the event was not
                                      a push to `main`.

  Docker build fails in CI            Required files may be missing from
                                      Git or excluded by `.dockerignore`.
  -----------------------------------------------------------------------

## 7. Best Practices

-   Keep tests repeatable and independent of the local machine.
-   Store credentials only in GitHub Secrets.
-   Match Python versions between CI and Docker.
-   Publish images only after tests pass.
-   Use commit-SHA tags for traceability.
-   Consider adding linting, vulnerability scanning, Docker layer
    caching, and a deployment job.
