# Company Landing Page

A responsive, professional company landing page built with plain HTML5 and CSS3.
It is packaged as a Docker image served by Nginx and deployed automatically
through a Jenkins declarative pipeline.

## Application Overview

The website contains these sections:

1. Navigation bar with the company name (collapses into a menu on mobile)
2. Hero section with heading, description and call-to-action button
3. About Us section
4. Services section with three service cards
5. Why Choose Us section
6. Contact section
7. Footer

There are no frameworks, no JavaScript dependencies, no backend, no database and no external APIs.
The design is responsive for desktop, tablet and mobile, with CSS hover effects.

## Project Structure

```
company-website/
├── index.html     # Page markup and all website sections
├── style.css      # Styling, hover effects and responsive media queries
├── Dockerfile     # Builds the Nginx image containing the website
├── Jenkinsfile    # Declarative CI/CD pipeline
└── README.md      # Project documentation
```

## How Docker Is Used

The `Dockerfile` uses the official `nginx:stable-alpine` image, copies `index.html`
and `style.css` into `/usr/share/nginx/html/`, and exposes container port 80.

Run locally:

```bash
docker build -t company-website .
docker run -d --name company-website -p 9090:80 company-website
```

Then open http://localhost:9090.

## Jenkins Deployment Process

The pipeline runs entirely on a Jenkins agent labeled `build-server`. The Jenkins controller
never builds images, and the build server never runs the application. The
container runs only on the deployment server.

| Stage | What happens |
|---|---|
| Fetch GitHub Repository | Clones the repository on the build server. |
| Build Docker Image | Builds the image, tagged with the build number and `latest`. |
| Login to Docker Hub | Logs in using Jenkins credentials (no secrets in the code). |
| Push Image to Docker Hub | Pushes both tags to Docker Hub. |
| Deploy on Deployment Server | Connects over SSH, pulls the image, stops and removes the old container, and starts the new one with `-p 9090:80`. |
| Verify Deployment | Confirms the container is running and responds with HTTP success on port 9090. |

After a successful run, the site is available at `http://DEPLOYMENT_SERVER_IP:9090`.

## Prerequisites

**Jenkins**
- An agent with the label `build-server` that has Docker and Git installed, with the Jenkins user allowed to run Docker.
- Plugins: Pipeline, Git, Credentials Binding, SSH Credentials.
- Credentials:
  - Docker Hub (Username with password, use an access token as the password)
  - Deployment server (SSH Username with private key)

**Deployment server**
- Docker installed.
- The SSH user can run Docker commands.
- Port 9090 open in the firewall or security group.
- The public key matching the Jenkins SSH credential is in `~/.ssh/authorized_keys`.
- `curl` installed (used by the verification step).
- If the Docker Hub repository is private, run `docker login` once on this server.

## Placeholders to Replace

All placeholders are in the `environment` block of the `Jenkinsfile`.

| Placeholder | Replace with |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account name |
| `DOCKERHUB_REPOSITORY` | Docker Hub repository name |
| `DOCKERHUB_CREDENTIAL_ID` | Jenkins Docker Hub credential ID |
| `DEPLOYMENT_SERVER_IP` | IP address of the deployment server |
| `DEPLOYMENT_SSH_CREDENTIAL_ID` | Jenkins SSH credential ID |
| `GITHUB_REPOSITORY_URL` | GitHub repository URL |

## Creating the Jenkins Job

1. Push this project to your GitHub repository.
2. In Jenkins, create a new **Pipeline** job.
3. Under Pipeline, select **Pipeline script from SCM**, choose Git, and enter the repository URL.
4. Set the script path to `Jenkinsfile` and the branch to `main`.
5. Click **Build Now**.

## Security Notes

- No passwords, tokens or private keys are stored in the repository.
- Secrets are supplied only through Jenkins Credentials at runtime.
