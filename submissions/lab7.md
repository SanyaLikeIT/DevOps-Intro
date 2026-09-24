# Lab 7 — Configuration Management: Deploy QuickNotes via Ansible

## Environment

- Host Python: 3.12.3
- Host Ansible distribution: 10.7.0 in pipx
- Host ansible-core: 2.17.14
- VM: Lab 5 Vagrant VirtualBox machine `quicknotes-vm`, x86_64, running
- SSH target: `vagrant@127.0.0.1:2222` with the Vagrant private key path in the inventory
- Guest ansible-pull: ansible-core 2.16.3

## Task 1 — Idempotent QuickNotes Deployment

### 1.1 Inventory

The [inventory](../ansible/inventory.ini) targets the actual Lab 5 Vagrant SSH endpoint. `ansible -i ansible/inventory.ini quicknotes -m ping` returned `pong`.

### 1.2 Playbook

The [playbook](../ansible/playbook.yaml) uses `become: true`, disables unused fact gathering, creates the system group and non-login system user, manages the data directory, binary, seed, unit and service with dedicated modules. The binary and unit template notify `restart quicknotes` only on change.

The [binary](../ansible/files/quicknotes) is a stripped, statically linked Linux x86-64 ELF built with `CGO_ENABLED=0` in the locally available Go 1.24 container. `ldd` reported `not a dynamic executable`. The copied [seed](../ansible/files/seed.json) passed `cmp` against `app/seed.json`.

### 1.3 Systemd template

The [template](../ansible/templates/quicknotes.service.j2) renders the user, group, paths, listen address and restart backoff from play variables.

### 1.4 Check mode

`ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml --check` succeeded before deployment:

```text
PLAY RECAP *********************************************************************
lab5-vm                    : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 1.5 First deployment

`ansible-playbook -i ansible/inventory.ini ansible/playbook.yaml` installed the files and ran the restart handler:

```text
TASK [Install QuickNotes binary] ***********************************************
changed: [lab5-vm]

TASK [Install seed notes] ******************************************************
changed: [lab5-vm]

TASK [Install QuickNotes systemd unit] *****************************************
changed: [lab5-vm]

TASK [Enable and start QuickNotes] *********************************************
ok: [lab5-vm]

RUNNING HANDLER [restart quicknotes] *******************************************
changed: [lab5-vm]

PLAY RECAP *********************************************************************
lab5-vm                    : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 1.6 Service verification

`systemctl is-active quicknotes` returned `active`; `systemctl is-enabled quicknotes` returned `enabled`. The process runs as `quicknotes`. `systemctl cat quicknotes` showed `User=quicknotes`, `ADDR=:8080`, `DATA_PATH=/var/lib/quicknotes/notes.json`, `SEED_PATH=/var/lib/quicknotes/seed.json`, and `RestartSec=2s`.

`stat` showed `/var/lib/quicknotes` as `quicknotes:quicknotes 750`, seed as `quicknotes:quicknotes 640`, and binary as `root:root 755`. The account shell is `/usr/sbin/nologin`.

### 1.7 Health and seeded notes

Both requests through the Vagrant forward returned HTTP 200.

`GET /health`:

```json
{"notes":4,"status":"ok"}
```

`GET /notes`:

```json
[{"id":1,"title":"Welcome to QuickNotes","body":"This is the project you'll containerize, deploy, monitor, and harden across all 10 labs.","created_at":"2026-01-15T10:00:00Z"},{"id":2,"title":"Read app/main.go first","body":"Start by understanding the entry point — env vars, signal handling, graceful shutdown.","created_at":"2026-01-15T10:05:00Z"},{"id":3,"title":"DevOps mantra","body":"If it hurts, do it more often.","created_at":"2026-01-15T10:10:00Z"},{"id":4,"title":"Endpoint cheat-sheet","body":"GET /notes  GET /notes/{id}  POST /notes  DELETE /notes/{id}  GET /health  GET /metrics","created_at":"2026-01-15T10:15:00Z"}]
```

The four note titles match the provided seed file.

### 1.8 Design questions

#### a) command vs modules

`command` and `shell` run commands but do not inherently understand the desired state of an account, file or service. `user`, `file`, `copy`, `template` and `systemd_service` compare the desired state with the current state and change only what differs. A command can be made idempotent with extra conditions, but these modules make repeated runs more predictable.

#### b) notify and handlers

The binary copy and unit template notify `restart quicknotes` only when they report `changed`. The handler runs once at the end of the play; unchanged tasks do not restart the service.

#### c) variable hierarchy

Role defaults would suit reusable low-precedence values; `group_vars/quicknotes.yml` would suit settings shared by this host group across plays; play vars keep this small lab's values beside the tasks that use them. The play uses play vars for clarity. Inventory retains connection settings.

#### d) gather_facts

This play needs no discovered OS facts, so `gather_facts: false` avoids running the initial setup module and collecting facts on every run.

## Task 2 — Idempotency and Selective Changes

### 2.1 Second run: changed=0

The unchanged second run did not invoke the handler:

```text
PLAY RECAP *********************************************************************
lab5-vm                    : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 2.2 Selective template change and handler

Changing only `listen_addr` from `:8080` to `:9090` left the user, group, directory, binary, seed and service management tasks `ok`. The template changed and notified the restart handler:

```text
TASK [Install QuickNotes systemd unit] *****************************************
changed: [lab5-vm]

TASK [Enable and start QuickNotes] *********************************************
ok: [lab5-vm]

RUNNING HANDLER [restart quicknotes] *******************************************
changed: [lab5-vm]
PLAY RECAP *********************************************************************
lab5-vm                    : ok=8    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The address was then restored to `:8080` with a real playbook run; `/health` returned HTTP 200.

### 2.3 --check --diff

With only `restart_sec` temporarily set to `3s`, `--check --diff` showed this actual proposed unit change:

```diff
--- before: /etc/systemd/system/quicknotes.service
+++ after: /home/alex/.ansible/tmp/ansible-local-506711hf7f12qd/tmpohzcaats/quicknotes.service.j2
@@ -13,7 +13,7 @@
 Environment="SEED_PATH=/var/lib/quicknotes/seed.json"
 ExecStart=/usr/local/bin/quicknotes
 Restart=on-failure
-RestartSec=2s
+RestartSec=3s

 [Install]
 WantedBy=multi-user.target
```

The value was returned to `2s` in the source playbook without applying `3s` to the VM.

### 2.4 Final restored configuration

The VM unit showed `ADDR=:8080` and `RestartSec=2s`; `/health` returned HTTP 200 and `/notes` still contained four seeded notes.

### 2.5 Design questions

#### e) Why changed=0

`group` and `user` found the existing account state; `file` found the intended directory and permissions; `copy` found the same content and metadata; `template` rendered the same unit; and `systemd_service` found the service already enabled and running. No task notified the handler.

#### f) Why not shell

A `shell: echo ... > /etc/systemd/system/quicknotes.service` task would normally run on every play, with fragile quoting and poor content diffs. Ansible could not compare the intended unit content or notify a restart only for a real change without extra logic. `template` supplies those comparisons and a clear diff. Shell can be guarded, but it is the wrong abstraction for this unit.

#### g) Why --check --diff

`--check` predicts changes; `--diff` shows supported content changes. The diff exposed the exact `RestartSec` edit, which a plain changed result would not show. The same check could reveal a wrong port, path, or removed unit line before deployment.

## Bonus — ansible-pull GitOps Loop

### B.1 Architecture and artifacts

The [bonus setup playbook](../ansible/bonus-pull.yaml) installs Git and ansible-core, copies the [local inventory](../ansible/files/local-inventory.ini) to `/etc/ansible/quicknotes-local.ini`, and installs the [service](../ansible/templates/ansible-pull.service.j2) and [timer](../ansible/templates/ansible-pull.timer.j2). The service pulls the actual public origin URL and checks out `feature/lab7`; the timer specifies `OnBootSec=1min`, `OnUnitActiveSec=5min`, and `Persistent=true`.

### B.2 Timer verification

The setup play recap was `ok=6 changed=6 failed=0`. `systemctl is-active ansible-pull.timer` returned `active`, and `is-enabled` returned `enabled`. `systemctl list-timers --all` showed:

```text
Thu 2026-09-24 20:52:37 UTC 4min 47s Thu 2026-09-24 20:47:37 UTC   12s ago ansible-pull.timer           ansible-pull.service
```

The installed `ansible-pull` is 2.16.3, and Git is 2.43.0.

### B.3 Initial pull result

At `2026-09-24T20:47:37+00:00`, the timer started `ansible-pull.service`. The public repository clone succeeded, but checkout of `feature/lab7` failed because the branch had not been pushed. The host's HTTPS push requested unavailable interactive credentials; its SSH GitHub key was rejected. The branch and timer demonstration require GitHub push access. The journal reported:

```text
2026-09-24T20:47:42+00:00 quicknotes-vm ansible-pull[4587]:     "msg": "Failed to checkout feature/lab7",
2026-09-24T20:47:42+00:00 quicknotes-vm ansible-pull[4587]:     "stderr": "error: pathspec 'feature/lab7' did not match any file(s) known to git\n",
2026-09-24T20:47:42+00:00 quicknotes-vm systemd[1]: ansible-pull.service: Failed with result 'exit-code'.
```

### B.4 Convergence timeline

Awaiting authenticated push of `feature/lab7`; no convergence time is claimed.

### B.5 Final restored state

The currently installed QuickNotes unit remains at `ADDR=:8080`, `RestartSec=2s` and serves four seeded notes. Timed pull convergence remains unproven.

### B.6 Design questions

#### h) Pull security

Push mode requires a control node with administrative access to managed hosts. Pull mode lets each node initiate outbound retrieval, reducing the need for inbound administrative access. Repository integrity, branch protection, and the software supply chain still matter; pull mode is not automatically secure.

#### i) Kubernetes equivalent

Argo CD and Flux are Kubernetes GitOps controllers. They repeatedly reconcile Git desired state with actual cluster state. `ansible-pull` applies the same Git → reconciliation → actual state loop to this VM.
