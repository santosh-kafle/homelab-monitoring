# Security

## Reporting a problem

Please **don't open a public issue** for a security problem. Report it privately on
GitHub instead: go to the **Security** tab of this repository and click
**Report a vulnerability**
([direct link](https://github.com/santosh-kafle/homelab-monitoring/security/advisories/new)).

Say what the problem is, how to reproduce it, and what someone could do with it. You'll
get an answer within a week. This is a hobby project run by one person, so a fix may take
a little longer, but you'll be kept up to date.

## Supported versions

Only the latest release gets fixes. Update with `./scripts/update.sh`.

## What counts

Things this project is responsible for, for example:

- A service listening on something other than `127.0.0.1` when it shouldn't.
- A script that could leak your Grafana password or ntfy topic, or put them in Git.
- A script deleting data without being asked to.
- A container getting more access to the host than the README says it has.

Problems inside Prometheus, Grafana, Node Exporter, cAdvisor or Alertmanager themselves
belong to those projects, please report them there. If one of them affects this stack and
a new version fixes it, an issue here asking to update is welcome.

## What this stack can see

So you know what you're running:

- **Node Exporter** reads your whole filesystem (read-only) and the host's network.
- **cAdvisor** reads `/var/lib/docker` (read-only) and talks to Docker's socket, so it
  can see every container's settings, including environment variables. Mounting the
  socket read-only doesn't limit what can be asked through it, so like anything with the
  socket, cAdvisor could control Docker. It only asks for container info. It can also
  read the kernel log.
- **Alertmanager** sends alert text (disk names, container names) to ntfy.sh if you turned
  on phone notifications. Nothing else leaves your machine.

Everything listens on `127.0.0.1` only, except Grafana if you opened it to your home
network. Never forward any of these ports to the internet.
