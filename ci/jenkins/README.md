# Local Jenkins for the jbfinance pipeline

Optional scaffolding for running the `Jenkinsfile` against a Jenkins controller
on your own machine. Delete this folder if you host Jenkins elsewhere.

## 1. Start the controller

```bash
docker compose -f ci/jenkins/docker-compose.yml up -d --build
docker exec jbfinance-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open http://localhost:8080, unlock with that password, and choose
**Select plugins to install > None** (the image already bundles `plugins.txt`).

## 2. Make it reachable from GitHub

GitHub.com must reach the controller for webhooks and the GitHub App. Use a tunnel:

```bash
# Cloudflare (no account needed for a quick tunnel)
cloudflared tunnel --url http://localhost:8080
# or: ngrok http 8080
```

Put the resulting `https://…` URL in:

- `docker-compose.yml` -> `JENKINS_URL`, then `docker compose … up -d` again
- **Manage Jenkins > System > Jenkins URL**

## 3. Tell the pipeline which system it is

**Manage Jenkins > System > Global properties > Environment variables**, add:

| Name          | Value     |
| ------------- | --------- |
| `CI_PROVIDER` | `jenkins` |

The CloudBees CI controller gets `CI_PROVIDER = cloudbees` instead, so the two
report distinct GitHub checks (`jenkins/ci` and `cloudbees/ci`).

## 4. Connect to GitHub + create the job

Follow the main runbook from **step 4** onward (GitHub App, credentials,
Multibranch Pipeline, branch protection).
