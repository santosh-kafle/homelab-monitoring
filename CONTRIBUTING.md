# Contributing

Thanks for wanting to help! This project is meant for people learning monitoring, so
questions and small fixes are just as welcome as big features. If you're not sure
whether something is a good idea, open an issue and ask first.

## Ways to help

- **Report a bug.** The bug report form asks for everything needed to look into it.
- **Try it on your hardware.** Did it work on a distro or machine that isn't mentioned in
  the README? An issue saying so helps the next person.
- **Fix the docs.** If something confused you, it confuses others too.
- **Add a panel or an alert.** See below for how.

## Setting up

You need the same things as for a normal install (see [Requirements](README.md#requirements)).

```bash
git clone https://github.com/<your-username>/homelab-monitoring.git
cd homelab-monitoring
./scripts/install.sh
```

Work on a branch, not on `master`:

```bash
git switch -c fix-battery-panel
```

## Making changes

### Dashboard panels

The dashboard is read-only in Grafana on purpose, the JSON file is the real source (see
[Everything is set up from files](README.md#everything-is-set-up-from-files)). Two ways
to change it:

- **Edit the JSON directly** in `grafana/dashboards/laptop-overview.json`, save, and
  refresh the browser. Grafana picks it up within 10 seconds.
- **Use Grafana's editor:** open the dashboard, click **Edit → Save as** to make your own
  copy, change it there, then **Export → Export as JSON** and copy the parts you changed
  back into the file.

Keep the data source as `{"type": "prometheus", "uid": "prometheus"}`, otherwise the panel
breaks on everyone else's install.

### Alert rules

Rules live in `prometheus/alerts.yml`. Before adding one:

1. Run the expression **without the threshold** in Prometheus (`http://127.0.0.1:9090`,
   Query tab) and check it returns one sensible series per thing you care about. Things
   like btrfs subvolumes or snap mounts easily turn one alert into eight.
2. Make it fire for real (fill a disk, stress the CPU, crash a container) and check it
   shows up at `http://127.0.0.1:9093`.
3. Add it to the alerts table in the README.

Apply changes with `docker compose up -d --force-recreate prometheus`.

### Scripts

The scripts in `scripts/` are bash, start with `set -euo pipefail`, and share helpers from
`scripts/lib.sh`. Messages are written for beginners: say what went wrong **and** what to
run to fix it.

### Things every change should keep true

- **Nothing listens on anything but `127.0.0.1`,** except Grafana when the user chose
  their home network.
- **No secrets in Git.** Passwords go in `.env`, notification secrets in
  `alertmanager/secrets/`. Both are ignored.
- **Image versions stay pinned,** no `:latest`.
- **Your data is never deleted** except by `uninstall.sh --remove-data`.
- **Comments say why, not what.** The README explains every decision, and new ones
  should be explained too.

## Checking your changes

CI runs these on every pull request, and you can run them yourself first. You don't need
to install anything, they run in Docker:

```bash
# scripts
docker run --rm -v "$PWD:/mnt:ro" -w /mnt koalaman/shellcheck:stable -x -P SCRIPTDIR scripts/*.sh

# prometheus config and alert rules
docker run --rm -v "$PWD/prometheus:/etc/prometheus:ro" --entrypoint promtool \
  prom/prometheus:v3.15.0 check config /etc/prometheus/prometheus.yml

# alertmanager config
docker run --rm -v "$PWD/alertmanager:/etc/alertmanager:ro" --entrypoint amtool \
  prom/alertmanager:v0.34.1 check-config /etc/alertmanager/alertmanager.yml

# compose file (the password is a dummy, compose just needs something)
GF_SECURITY_ADMIN_PASSWORD=unused docker compose config --quiet
```

Use the image versions from `docker-compose.yml` if they've changed since this was
written.

CI also does a full `install.sh --yes`, then `uninstall.sh --remove-data --yes`, and
checks nothing was left behind.

## Pull requests

- Keep one change per pull request, it's easier to review and to undo.
- If users will notice the change, add a line under `[Unreleased]` in
  [CHANGELOG.md](CHANGELOG.md), written for them, not for developers.
- Say how you tested it: which distro, which machine, what you saw.

## Releases

The maintainer moves `[Unreleased]` into a new version in the changelog and tags it
(`v0.5.0`). Version numbers follow [Semantic Versioning](https://semver.org/). Published
tags are never moved, because `update.sh` relies on them.
