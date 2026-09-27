# QuickNotes in GitHub Codespaces

Render requested card verification, so no payment method was added. Lab 10's Codespaces Option B is used.

The devcontainer configuration is [`.devcontainer/devcontainer.json`](../.devcontainer/devcontainer.json). It uses the official Ubuntu devcontainer base, Docker-in-Docker and SSH features, forwards TCP port 8080, and runs [the startup script](../.devcontainer/start-quicknotes.sh) on each Codespace start. The SSH feature enables the GitHub CLI to inspect the running container and collect logs.

The script pulls and runs `ghcr.io/sanyalikeit/devops-intro/quicknotes:v0.1.1`. It checks the container's image ID, starts an existing stopped container when possible, and recreates it only if the release image differs. It maps port 8080, sets `ADDR=0.0.0.0:8080`, `DATA_PATH=/data/notes.json`, and `SEED_PATH=/app/seed.json`. QuickNotes runs as UID 65532, so the persistent host directory `/workspaces/.quicknotes-lab10-data` is owned by that UID and mounted at `/data`. The runtime notes file stays outside the Git repository.

Port forwarding is declared in the devcontainer; public visibility must be set separately after creation. The creation, port visibility, public URL, stop/start commands, and measured behavior are recorded in [the submission](../submissions/lab10.md) after the live deployment.
