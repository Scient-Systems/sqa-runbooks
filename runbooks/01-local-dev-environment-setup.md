# 01 — Local dev environment setup (one-time)

Goal: a working WSL2 + Docker + Tutor environment with all repos cloned, ready to run Open edX
locally. Do this once per machine. Budget ~2–3 hours (most of it is the first image build,
which runs unattended).

> **Why WSL2 and not native Windows?** Tutor on native Windows is broken — paths, permissions,
> and inotify (file-watching for hot reload) all fail in subtle ways. `tutor dev` (the only mode
> with live code mounting) requires Linux. We use **WSL2 + Ubuntu**, with all source **inside the
> Linux filesystem** (`~/...`), never under `/mnt/c/...` (that path kills inotify and rebuild speed).

---

## 1. WSL2 + Ubuntu

In an **elevated PowerShell** on Windows:

```powershell
wsl --install -d Ubuntu-22.04
```

Reboot if prompted, then launch Ubuntu and set your username + password. Verify from PowerShell:

```powershell
wsl --list --verbose      # Ubuntu-22.04 should be STATE=Running, VERSION=2
```

> If `wsl --install` hangs: manually enable the Windows features "Virtual Machine Platform" and
> "Windows Subsystem for Linux", reboot, retry.

## 2. Docker Desktop

Install Docker Desktop for Windows. Then **Settings → Resources → WSL Integration → enable for
Ubuntu-22.04**. Verify inside the **Ubuntu shell**:

```bash
docker ps          # must work with NO sudo and NO permission error
```

> If Docker can't see the distro, restart Docker Desktop after toggling the integration.
>
> **Disk note:** Docker images live on Docker Desktop's data disk (default C:). The LMS image
> alone is ~4 GB. If C: is tight, move it: Docker Desktop → Settings → Resources → Advanced →
> Disk image location, *before* the first build.

## 3. System packages + Python

Ubuntu 22.04 ships Python 3.10 — that is fine for our plugin (`python_requires=">=3.8"`).

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv git make
```

## 4. Install Tutor (inside WSL)

```bash
pip install --user "tutor[full]"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
tutor --version          # should print 21.x
```

> 🚨 Tutor lives in `~/.local/bin`. If you ever run Tutor from a *non-login* shell
> (e.g. `wsl -e bash -c "..."`), that dir isn't on PATH and `tutor` won't be found. Always use a
> **login shell**: `wsl -e bash -lc "..."` (note the `-l`).

## 5. Clone every repo — INSIDE the Linux filesystem

```bash
cd ~
git clone https://github.com/Scient-Systems/sqa_django_app.git
git clone https://github.com/Scient-Systems/frontend-app-sqa-payment.git
git clone https://github.com/Scient-Systems/tutor-indigo.git
git clone https://github.com/Scient-Systems/brand-openedx.git
git clone https://github.com/Scient-Systems/sqa-homepage.git
git clone https://github.com/Scient-Systems/sqa-proxytoken-service.git
# course content lives in ~/silly_bot_course_project
```

> 🚨 **Never** clone under `/mnt/c/...`. Confirm with `ls ~/sqa_django_app/SQA_PLUGIN_CONTEXT.md`.
> Several repos are **private** — you'll need a GitHub Personal Access Token (PAT) with `repo`
> scope. Store it via `git config --global credential.helper store` then clone once with the PAT;
> it lands in `~/.git-credentials`. For pushing branches that touch `.github/workflows/*` the PAT
> also needs **workflow** scope (classic token). SSH is *not* set up on this machine.

## 6. Generate the initial Tutor config

```bash
tutor config save
tutor config printroot     # should print a path INSIDE WSL, e.g. /home/<you>/.local/share/tutor
```

## 7. Install the Django plugin for local (non-container) testing

This lets you run the fast `pytest` tier without any LMS. (Container install is a separate step
covered in runbook 03/06.)

```bash
cd ~/sqa_django_app
pip install -e ".[test]" --timeout 120
pytest tests/ -v           # should pass
```

### 🚨 pip gotchas we hit (all already fixed in the repo, listed so you recognize them)

- `opaque-keys` doesn't exist on PyPI — the real name is **`edx-opaque-keys`**.
- `Django` in `requirements/base.in` MUST be pinned (`Django>=4.2,<5.3`). Unpinned, pip backtracks
  through every Django release ever and takes 30+ minutes.
- The cookiecutter default `python_requires=">=3.12"` blocks install on Python 3.10 — we lowered it
  to `>=3.8`.
- **`edx-opaque-keys==2.12.0` locally.** Newer (`>=2.13.0`) uses `typing.Self` (Python 3.11+) and
  breaks on local 3.10: `pip install "edx-opaque-keys==2.12.0"`. 🚨 **Do NOT pin this in
  `requirements/constraints.txt`** — that pin would then apply *inside the LMS container* (Python
  3.11) and downgrade the LMS's opaque-keys, breaking LMS startup with
  `ImportError: cannot import name 'CollectionKey'`.

## 8. Verify the whole stack in one line

```bash
wsl -e bash -lc "docker ps >/dev/null && tutor --version && test -f ~/sqa_django_app/SQA_PLUGIN_CONTEXT.md && echo OK"
```

Prints the Tutor version and `OK` when the machine is ready.

---

## What's next

- Understand what Tutor actually is → **02-tutor-architecture-explained.md**
- First-time build + run the platform → **03-running-openedx-locally.md** (the first
  `tutor dev launch` builds the LMS image from scratch — 1–2 hours, unattended)
