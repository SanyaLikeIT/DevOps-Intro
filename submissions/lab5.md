# Lab 5 — Virtualization: QuickNotes in a Vagrant VM

## Environment

- Host OS: Ubuntu 24.04.4 LTS, x86_64, kernel `7.0.0-34-generic`.
- Docker: `29.6.1`.
- rsync: `3.2.7`.
- VM box configured: `bento/ubuntu-24.04` (execution pending).
- Go configured: `1.24.5`, with the official archive SHA256 checked during provisioning.

## Task 1 — Vagrant Up and QuickNotes

### 1.1 Vagrantfile

The root [Vagrantfile](../Vagrantfile) configures a two vCPU, 1024 MB Ubuntu 24.04 VM named `quicknotes-vm`. It syncs `./app` to `/opt/quicknotes` with rsync, builds QuickNotes, and enables a systemd service. The service stores writable notes in `/var/lib/quicknotes`, separate from the synced source. The forwarded port is `127.0.0.1:18080` on the host to port 8080 in the guest.

### 1.2 Provisioning and `vagrant up`

Pending installation of VirtualBox and Vagrant on the host. No successful VM boot has been recorded yet.

### 1.3 Guest verification

Pending a running VM.

### 1.4 Host verification

Pending a running VM.

### 1.5 Design questions

#### a) Synced folders

I chose rsync in the Vagrantfile because it is simple, reliable for a small source tree, and does not depend on matching VirtualBox Guest Additions for shared folders. The host source is authoritative. It is not a live bidirectional mount: after changing files on the host, I must run `vagrant rsync` (or trigger another sync) to update the guest.

#### b) NAT, bridged, and host-only networking

The VM uses Vagrant's default NAT network with one explicit forwarded port: `127.0.0.1:18080` to guest port 8080. Binding to loopback allows the host to access the service without putting the guest directly on the local network. A bridged adapter would give the VM broader LAN presence. Host-only networking could also isolate the guest, but an explicit loopback forward is enough for this exercise.

#### c) Provisioning

I used shell provisioning because this bootstrap installs one pinned toolchain, builds one service, and needs no extra configuration-management dependency. The shell script checks the installed Go version, so a later `vagrant provision` does not download Go again when it is already correct. Ansible can take over configuration management in Lab 7.

#### d) Go point release

`1.24` identifies a series whose latest patch can change. Pinning `1.24.5` gives students the same toolchain during rebuilds and makes behavior and troubleshooting more predictable.

## Task 2 — Snapshots

The snapshot lifecycle is pending a running VM.

### Snapshot design questions

#### e) Why snapshots are not backups

A snapshot depends on the VM disk chain and host storage that contain it. It cannot recover the VM after loss of the host disk, corruption or deletion of the VM files, or loss of the machine holding both the base image and snapshots. An independent backup must live separately.

#### f) Copy-on-write

A snapshot initially references existing disk blocks and stores later changes separately. Ten snapshots therefore do not immediately require ten complete copies of the VM disk. Changed blocks and metadata accumulate as the chain grows.

#### g) Snapshot antipattern

Long snapshot chains consume increasing storage and make management and restores more complex. They are a poor substitute for reproducible provisioning and independent backups.

## Bonus — VM vs Docker

### Docker measurements

The same source from `./app` was run in `golang:1.24` as `lab5-quicknotes`, with host port `127.0.0.1:28080` forwarded to port 8080. The container returned `HTTP/1.1 200 OK` and `{"notes":8,"status":"ok"}` from `/health`.

After `docker stop lab5-quicknotes`, `/usr/bin/time -p docker start lab5-quicknotes` reported:

```text
real 0.37
user 0.05
sys 0.03
```

An immediate curl initially got `Recv failure: Connection reset by peer`; a retry returned the healthy response. The start time above measures the Docker command, not full HTTP readiness.

`docker stats --no-stream lab5-quicknotes` reported `6.84MiB / 15.32GiB` memory usage. `docker top lab5-quicknotes` showed two process rows: the shell command and `/tmp/qn`. `docker inspect` identified the configured image as `golang:1.24`, and `docker images` reported its size as `894MB`.

### Comparison

VM values are pending a successful boot and direct measurements on this host.
