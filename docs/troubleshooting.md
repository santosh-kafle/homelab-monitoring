# Troubleshooting

Find your problem below. If it's not here, open a
[bug report](https://github.com/santosh-kafle/homelab-monitoring/issues/new/choose).

- [The installer stops with a list of problems](#the-installer-stops-with-a-list-of-problems)
- [Some panels are empty](#some-panels-are-empty)
- [Everything is empty](#everything-is-empty)
- [A container is missing from the Containers row](#a-container-is-missing-from-the-containers-row)
- [Grafana says my password is wrong](#grafana-says-my-password-is-wrong)
- [I can't open Grafana from another device](#i-cant-open-grafana-from-another-device)
- [I don't get notifications on my phone](#i-dont-get-notifications-on-my-phone)
- [I changed a config file and nothing happened](#i-changed-a-config-file-and-nothing-happened)
- [Grafana won't let me save the dashboard](#grafana-wont-let-me-save-the-dashboard)
- [Monitoring stopped after a reboot](#monitoring-stopped-after-a-reboot)
- [update.sh refuses to update](#updatesh-refuses-to-update)
- [How much disk space does this use?](#how-much-disk-space-does-this-use)

## The installer stops with a list of problems

That's on purpose. It checks everything first and lists **all** the problems at once,
each with the command that fixes it. Fix them, run `./scripts/install.sh` again. Nothing
was changed on your machine.

The two most common:

- **"you dont have permission to use docker"**: run
  `sudo usermod -aG docker $USER`, then **log out and back in**. Opening a new terminal
  isn't enough. `groups` should list `docker` afterwards.
- **"port … is already in use"**: another program has that port. The message shows how to
  find which one. For Grafana's port you can pick another one instead: add
  `GRAFANA_PORT=3001` to `.env`.

## Some panels are empty

Usually that's your hardware, not a bug. See
[Some panels are empty](../README.md#some-panels-are-empty-is-that-broken) in the README:
no battery on a desktop, no sensors in a virtual machine, and so on.

Also check the time range in the top right of Grafana. Right after installing there's only
a few minutes of data, so "Last 7 days" looks almost empty.

## Everything is empty

Check each piece from the bottom up. Whichever step fails first is the broken one:

```bash
docker compose ps                                     # 0. are all 5 containers "Up"?
curl -s 127.0.0.1:9100/metrics | head -3              # 1. is Node Exporter collecting?
curl -s 127.0.0.1:9338/healthz                        # 2. is cAdvisor up? says "ok"
curl -s 127.0.0.1:9090/-/ready                        # 3. is Prometheus up?
curl -s 127.0.0.1:9090/api/v1/targets | grep -o '"health":"[a-z]*"'   # 4. three times "up"?
curl -s 127.0.0.1:3000/api/health                     # 5. is Grafana up?
```

When a step fails, look at that service's log:

```bash
docker compose logs --tail 50 prometheus    # or node-exporter, cadvisor, grafana, alertmanager
```

A target that's **down** in step 4 shows why on `http://127.0.0.1:9090/targets`: hover over
the error.

If a container isn't behaving the way the config says, check what it actually got. The
file and the running container can drift apart:

```bash
docker inspect grafana --format '{{range .Mounts}}{{.Destination}}{{println}}{{end}}'
docker inspect node-exporter --format '{{range .Config.Cmd}}{{println .}}{{end}}'
```

## A container is missing from the Containers row

- **It only just started.** Give it a minute, cAdvisor and Prometheus each check every
  15 seconds.
- **It was started by Podman or rootless Docker.** cAdvisor only sees containers of the
  normal Docker on this machine.
- **It's been stopped.** Stopped containers disappear from the graphs, their old data stays.

## Grafana says my password is wrong

Get the password from `.env`:

```bash
grep GF_SECURITY_ADMIN_PASSWORD .env
```

Still wrong? Grafana only reads that password **the first time it starts**. After that it
keeps its own copy in its database. So if you changed `.env`, or deleted `.env` and ran the
installer again while the data was kept, the two no longer match. Set a new one directly
in Grafana:

```bash
docker exec -it grafana grafana cli admin reset-admin-password <new-password>
```

## I can't open Grafana from another device

1. **Is it open to the network at all?** `grep GRAFANA_BIND_ADDR .env` must say `0.0.0.0`.
   If not, set it and run `docker compose up -d`.
2. **Is a firewall blocking it?** Most likely on Fedora (firewalld) and Ubuntu with ufw on.
   The installer printed the command, or see
   [Opening Grafana to your home network](../README.md#opening-grafana-to-your-home-network).
3. **Are you using the right address?** Use this machine's IP on your home network, not
   `127.0.0.1`. To see it: `ip route get 1.1.1.1` (the address after `src`).
4. **Are both devices on the same network?** A guest Wi-Fi often blocks devices from
   seeing each other.

Only Grafana can be opened to the network. Prometheus, Alertmanager and the others always
stay private to the machine, on purpose.

## I don't get notifications on my phone

Go through these in order:

1. **Are notifications turned on?** `ls alertmanager/secrets/` must show `ntfy_url`. If not,
   run `./scripts/install.sh --ntfy`.
2. **Is your phone subscribed to the right topic?** Compare the topic in the app with
   `cat alertmanager/secrets/ntfy_url` (the part after `ntfy.sh/`, before the `?`).
3. **Is anything firing?** `http://127.0.0.1:9093` shows what Alertmanager has right now.
   If it's empty, there's nothing to send. Send a test alert:

   ```bash
   docker exec alertmanager amtool alert add TestAlert severity=warning \
     --annotation='summary="Test alert"' --alertmanager.url=http://127.0.0.1:9093
   ```

   It arrives after about 30 seconds.
4. **Did sending fail?** Look for errors:

   ```bash
   docker compose logs alertmanager | grep -i "notify"
   ```

   - `read url_file: ... no such file or directory`: notifications aren't turned on, see 1.
   - `connection refused`, `timeout`, `no such host`: this machine can't reach ntfy.sh.
     Check the internet connection, and any VPN or DNS blocker.
5. **Is the app allowed to notify?** On Android, check battery optimisation isn't putting
   ntfy to sleep. In the app, the topic shouldn't be muted.

You get one message when an alert starts, a reminder every 6 hours while it's still
firing, and one when it's resolved. Not one every few minutes, that's on purpose.

## I changed a config file and nothing happened

Each piece needs a different nudge:

| You changed | Run |
| --- | --- |
| `.env` | `docker compose up -d` |
| `docker-compose.yml` | `docker compose up -d` |
| `prometheus/prometheus.yml` or `prometheus/alerts.yml` | `docker compose up -d --force-recreate prometheus` |
| `alertmanager/alertmanager.yml` | `docker compose up -d --force-recreate alertmanager` |
| `alertmanager/secrets/ntfy_url` | Nothing, it's read every time an alert is sent |
| The dashboard JSON | Nothing, refresh the browser after 10 seconds |

Why `--force-recreate` and not `restart`? Single files are mounted into the containers.
Many editors save by writing a new file and swapping it in, and a running container keeps
looking at the old one. Recreating the container picks up the new file.

Check Prometheus actually loaded your rules on `http://127.0.0.1:9090/rules`. If there's a
typo, `docker compose logs prometheus` says which line.

## Grafana won't let me save the dashboard

On purpose: the dashboard comes from `grafana/dashboards/laptop-overview.json`, and that
file is the real source. Changes made only in Grafana would be lost on the next update.
Edit the file instead, or see
[CONTRIBUTING.md](../CONTRIBUTING.md#dashboard-panels) for how to use Grafana's editor and
copy the result back.

If you just want your own version, use **Save as** to make a copy. The copy is yours and
can be edited freely.

## Monitoring stopped after a reboot

The containers start again on their own, but only if Docker starts at boot:

```bash
sudo systemctl enable --now docker
```

## update.sh refuses to update

- **"you changed some of the project's files"**: you edited a file that's part of the project. Updating
  would overwrite it. See what you changed with `git diff`, then either undo it
  (`git checkout -- <file>`) or put it aside (`git stash`, and `git stash pop` after
  updating).
- **"couldnt reach github"**: check your internet connection, `git fetch` needs it.
- **There's no `scripts/update.sh` at all**: you're on v0.1.0, which didn't have it yet.
  Update once by hand, see the [Update](../README.md#update) section.

`git status` saying "HEAD detached at v0.4.0" after updating is normal: you're on an
exact release instead of a branch.

## How much disk space does this use?

- **Images:** about 2.5 GB on disk, most of it Grafana. Updates download new versions and
  keep the old ones. `docker image ls` lists them, and
  `docker image rm grafana/grafana:<old version>` removes one. (`docker image prune` won't,
  it only removes images without a version tag.)
- **Metrics:** Prometheus keeps 30 days, and never more than 5 GB. A single laptop
  usually needs a few hundred MB for those 30 days.
- **Grafana's database:** a few hundred MB.

To see the real numbers:

```bash
docker system df -v | grep homelab-monitoring
```
