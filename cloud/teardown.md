# Lab 10 teardown

After collecting and grading the evidence:

1. Stop the local container with `docker stop quicknotes-lab10-tunnel` and remove it with `docker rm quicknotes-lab10-tunnel`.
2. Stop the quick tunnel with `kill <recorded cloudflared PID>` or Ctrl+C in its terminal.
3. Optionally remove `/tmp/lab10-hf-warm.txt`, `/tmp/lab10-hf-cold*.txt`, `/tmp/lab10-tunnel-50.txt`, and `/tmp/lab10-cloudflared.log`.
4. Optionally delete the `quicknotes-lab10` Space from its Hugging Face repository settings after grading.
5. Optionally remove the local image with `docker image rm ghcr.io/sanyalikeit/devops-intro/quicknotes:v0.1.0 ghcr.io/sanyalikeit/devops-intro/quicknotes:latest` when no container uses it.
