Setting Up a CI Pipeline with GitHub Actions
A record of how I set up a Continuous Integration (CI) pipeline for my Flask application: what I did, why I did it, and how each part works.
1. Overview
A CI pipeline is an automated process that compiles code, installs application dependencies, and runs automated tests every time a developer pushes or pulls code to a shared repository. Its purpose is to catch broken code early, before it is merged or deployed.
For my Flask application, the pipeline does two things in order:
1. Runs the automated tests.
2. Builds a Docker image and publishes it to Docker Hub, but only if the tests pass.
I used GitHub Actions as the CI platform because my code is already hosted on GitHub and the workflows live in the same repository as the code.
Key terms
Term
Meaning
Workflow
An automated process defined in a YAML file inside .github/workflows/.
Event / trigger
Something that starts a workflow, such as a push or a pull request.
Job
A group of steps that run on the same machine. Jobs run in parallel unless one depends on another.
Step
A single task inside a job: either a shell command (run) or a reusable action (uses).
Runner
The machine that executes a job. I used GitHub's hosted ubuntu-latest runner.
Action
A reusable, prebuilt step, for example actions/checkout.
Secret
An encrypted value stored in the repository settings, used for credentials.
Artifact
The output of a build. In my pipeline, this is the Docker image.
2. Prerequisites
Before writing the pipeline I made sure of the following:
1. The application runs correctly on my local machine.
2. The application is containerized with a working Dockerfile (see my Docker setup guide).
3. The code is in a GitHub repository.
4. I have a Docker Hub account and a repository on it to publish the image to.
3. Steps I Followed
3.1 Understand what the application needs to run
I first listed what the Flask application requires: the Python version, the packages in requirements.txt, and any environment variables needed to run the tests. The pipeline starts on a clean machine, so it has to install and configure everything from scratch. Anything that works locally only because of something already installed on my laptop will fail in the pipeline.
3.2 Confirm the application works locally
I confirmed that the application installs and starts on my machine from a fresh virtual environment, so that I knew a pipeline failure would point to a pipeline problem and not an application problem.
3.3 Confirm all tests pass locally
I ran the full test suite locally and fixed any failing tests. A pipeline that fails on every run from the beginning is not useful, and it is much quicker to debug tests on my own machine than through repeated pushes.
pip install -r requirements.txt
pytest
3.4 Identify the jobs the pipeline should run
I decided on two jobs:
Job
Purpose
Runs when
test
Install dependencies and run the automated tests.
On every push and pull request.
build-and-push
Build the Docker image and publish it to Docker Hub.
Only after test passes, and only on pushes to main.
Splitting them means a broken change never produces a published image, and pull requests get test feedback without publishing anything.
4. Store Credentials as Repository Secrets
Publishing to Docker Hub requires authentication, and credentials must never be written into the workflow file or committed to the repository.
1. On Docker Hub, I created an access token (Account Settings, then Security, then New Access Token) with permission to read and write. I used a token rather than my account password so it can be revoked without changing my password.
2. In my GitHub repository I went to Settings, then Secrets and variables, then Actions, then New repository secret and added two secrets:
Secret name
Value
DOCKERHUB_USERNAME
My Docker Hub username.
DOCKERHUB_TOKEN
The access token created above.
In the workflow these are read with ${{ secrets.DOCKERHUB_USERNAME }} and ${{ secrets.DOCKERHUB_TOKEN }}. GitHub masks their values in the logs.
5. Create the Workflow File
I created the file .github/workflows/ci.yml at the root of the repository. GitHub automatically detects any YAML file in the .github/workflows/ directory.
name: CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out the code
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
      - name: Check out the code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push the image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app-image:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/my-app-image:${{ github.sha }}
I replaced my-app-image with the name of my own Docker Hub repository.
5.1 Section-by-section explanation
• name is the label that appears in the Actions tab of the repository.
• on defines the triggers. The pipeline runs on every push to main and on every pull request that targets main.
• jobs.test runs on a fresh Ubuntu runner:
    ◦ actions/checkout downloads my repository code onto the runner. Without it the runner has no code.
    ◦ actions/setup-python installs the Python version I specified. I matched it to the version in my Dockerfile so the tests run on the same version as production. cache: pip reuses downloaded packages between runs to speed things up.
    ◦ The run steps install dependencies and execute the tests. If any command exits with an error, the step fails, the job fails, and the pipeline stops.
• needs: test makes build-and-push wait for test and skip entirely if it fails.
• if: ... restricts publishing to pushes on main, so pull requests only run the tests.
• docker/login-action authenticates to Docker Hub using my secrets.
• docker/setup-buildx-action sets up Docker Buildx, the build engine used by the build action.
• docker/build-push-action builds the image from my Dockerfile and pushes it. context: . means the repository root, the same as the . in docker build.
• Two tags are pushed. latest always points to the newest build. The commit SHA tag (github.sha) uniquely identifies the image for every commit, so I can trace any image back to the exact code and roll back to a specific version.
6. Trigger and Watch the Pipeline
1. I committed and pushed the workflow file to GitHub:
git add .github/workflows/ci.yml
git commit -m "Add CI pipeline"
git push origin main
2. In the repository I opened the Actions tab, selected the CI Pipeline workflow run, and watched each job and step execute. Clicking a step shows its full log output.
3. A green check mark means every job succeeded. A red cross means a step failed, and the log shows which one and why.
7. Verify the Result
When both jobs passed, I confirmed the image was published:
1. On Docker Hub, I opened my repository and checked that the latest tag and the commit SHA tag were present, with a recent push time.
2. I pulled and ran the published image on my machine to make sure it works:
docker pull <dockerhub-username>/my-app-image:latest
docker run -p 5000:5000 <dockerhub-username>/my-app-image:latest
8. Troubleshooting
Symptom
Likely cause and fix
Workflow does not start
The file is not in .github/workflows/, has a YAML syntax error, or the branch does not match the on trigger.
Tests pass locally but fail in the pipeline
A dependency missing from requirements.txt, a different Python version, a missing environment variable, or a test that depends on local files or state.
ModuleNotFoundError in the test job
The package is not listed in requirements.txt.
Error: Cannot perform an interactive login or unauthorized on push
Secret names are misspelled, the secrets are missing, or the Docker Hub token lacks write permission.
denied: requested access to the resource is denied
The image name does not match my Docker Hub username and repository name.
build-and-push job is skipped
test failed, or the run was a pull request or a push to a branch other than main.
Image builds locally but not in the pipeline
A file excluded by .dockerignore or not committed to git that the Dockerfile needs.
Pipeline is slow
Enable dependency caching and make sure Dockerfile layers are ordered for caching.
9. Best Practices I Followed
1. Keep the pipeline fast so that feedback arrives within minutes.
2. Make tests deterministic: no reliance on my local machine, network services, or fixed order.
3. Store all credentials as secrets and use access tokens instead of passwords.
4. Pin the major versions of actions (for example @v4).
5. Only publish artifacts from the main branch, and only after tests pass.
6. Tag images with the commit SHA, not only latest, so builds are traceable.
7. Match the Python version in the pipeline to the one in the Dockerfile.
8. Treat a failing pipeline as the top priority to fix.
10. Possible Improvements
• Add a linting step (such as flake8 or ruff) before the tests.
• Run the tests against several Python versions using a build matrix.
• Cache Docker layers between builds to speed up image builds.
• Scan the built image for vulnerabilities before publishing.
• Add a deployment job (continuous delivery) that pulls the new image to a server after publishing.
• Add a status badge for the workflow to the project README.
11. Quick Reference
Workflow file:   .github/workflows/ci.yml
Secrets needed:  DOCKERHUB_USERNAME, DOCKERHUB_TOKEN
Job 1 (test):    checkout -> setup Python -> install dependencies -> run tests
Job 2 (publish): needs test; main only -> login -> build image -> push to Docker Hub
Tags pushed:     latest and the commit SHA
12. Final Checklist
[x] Application runs locally.
[x] All tests pass locally.
[x] Dockerfile builds a working image.
[x] Docker Hub access token created and stored as a GitHub secret.
[x] Workflow file created in .github/workflows/.
[x] Tests run automatically on every push and pull request.
[x] Image is built and pushed to Docker Hub only after tests pass.
[x] Published image pulled and verified.