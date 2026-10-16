# DevOps Notes

My personal notes and step-by-step guides on the DevOps tools I am learning. Each guide documents what I actually set up, why I made each choice, and what went wrong along the way.

The working examples (Dockerfile, CI workflow) live in my Flask application repository: **[my-flask-app](https://github.com/elvinkevin/secweb)**. These notes explain the reasoning; that repo contains the real, running configuration.

## Guides

| Topic | Guide | What it covers |
|---|---|---|
| Docker | [Containerizing an application](Docker/containerizing-an-app.md) | Writing a Dockerfile, `.dockerignore`, building images, running containers, troubleshooting. |
| CI/CD | [CI pipeline with GitHub Actions](ci-cd/CI-github-actions-pipeline.md) | Running tests on every push, building a Docker image, and publishing it to Docker Hub. |

## How the pieces fit together

```text
Code push to GitHub
        |
        v
GitHub Actions: run tests  --(fail)--> pipeline stops
        |
      (pass)
        v
Build Docker image from the Dockerfile
        |
        v
Publish image to Docker Hub
```

The Docker guide explains how the image is built; the CI guide explains how that build is automated.

## Repository structure

```text
devops-notes/
├── README.md
├── docker/
│   └── containerizing-an-app.md
└── ci-cd/
    └── github-actions-pipeline.md
```

Each new topic gets its own folder with a guide inside, and a new row in the table above.

## Tools used so far

- **Application:** Python / Flask
- **Containers:** Docker
- **CI platform:** GitHub Actions
- **Image registry:** Docker Hub

## Planned topics

- [ ] Docker Compose (running the app with a database)
- [ ] Deploying the published image to a server
- [ ] Linting and security scanning in the pipeline

## Conventions

- Every guide follows the same structure: overview, prerequisites, steps, verification, troubleshooting, best practices, checklist.
- Guides are updated whenever my setup changes, so they stay accurate.
- No credentials or secrets are ever included in these notes. Values like usernames and tokens are shown as placeholders.
