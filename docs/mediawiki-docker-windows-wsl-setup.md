# MediaWiki Development Environment on Windows — WSL + Docker Setup

A step-by-step walkthrough for getting MediaWiki core running locally on Windows using **WSL 2 + Docker Desktop**, which is what the official `DEVELOPERS.md` recommends for Windows users. Follow these steps in order; each builds on the previous. Every command shown is meant to be typed into the shell indicated at the start of the step — watch the shell prompts carefully, they change.

> Sources consulted while writing this: https://www.mediawiki.org/wiki/MediaWiki-Docker and the official `DEVELOPERS.md` at https://gerrit.wikimedia.org/g/mediawiki/core/+/HEAD/DEVELOPERS.md. Where instructions differ between documents, `DEVELOPERS.md` wins (it's the canonical source that ships with the codebase).

---

## Why this setup

Windows + Docker + a Windows-filesystem project folder is **very** slow for MediaWiki because Docker has to translate every filesystem call between Windows (NTFS) and Linux (ext4). Running the files inside WSL's native Linux filesystem removes that translation. The MediaWiki containers still run under Docker Desktop; they just mount files from inside WSL instead of from `C:\Users\…`.

**Mental model:**
```
Your code lives at      \\wsl.localhost\Ubuntu\home\<you>\mediawiki   (= /home/<you>/mediawiki inside WSL)
Docker Desktop runs on  Windows, but uses its WSL 2 backend
Containers mount        the WSL path above (fast), not a C:\ path (slow)
You can edit files from VS Code (WSL extension), File Explorer, or inside WSL itself — all see the same files.
```

---

## Step 1 — Enable WSL and install Ubuntu

Open **PowerShell as Administrator** (right-click Start → Terminal (Admin) or Windows PowerShell (Admin)).

```powershell
wsl --install -d Ubuntu
```

On a fresh Windows install this enables the "Virtual Machine Platform" and "Windows Subsystem for Linux" Windows features, then installs Ubuntu. **Reboot when prompted.**

After reboot, Ubuntu will launch automatically (or you can launch it from the Start menu). The first launch asks you to create a Linux username and password — write this down; you'll need it. **These are independent of your Windows credentials.**

Once Ubuntu is up and you're at its bash prompt, close the window and go back to PowerShell (Admin).

Make sure Ubuntu is running on WSL version 2, not 1:

```powershell
wsl --set-version Ubuntu 2
wsl --set-default-version 2
wsl -l -v
```

The output of `wsl -l -v` should show `Ubuntu … Running … 2`. If it shows 1, the `--set-version Ubuntu 2` command will convert it (takes a few minutes).

---

## Step 2 — Install Docker Desktop and enable the WSL 2 backend

1. Download Docker Desktop for Windows from https://www.docker.com/products/docker-desktop/ and install with the default options (includes the WSL 2 backend installer).
2. Launch Docker Desktop. Accept the license.
3. Go to **Settings (gear icon)** → **General** → check **"Use the WSL 2 based engine"**. Click **Apply & restart**.
4. Go to **Settings** → **Resources** → **WSL Integration** → toggle **"Enable integration with my default WSL distro"** ON and also tick **Ubuntu** under the distro list. Click **Apply & restart**.

Verify from PowerShell:

```powershell
docker --version
docker run --rm hello-world
```

The `hello-world` image should download and print a success message.

Also verify Docker is visible from inside WSL. Open Ubuntu (Start menu → Ubuntu) and in the WSL bash prompt run:

```bash
docker --version
docker run --rm hello-world
```

Both should work. If Ubuntu's `docker` command isn't found, the WSL Integration toggle in step 4 isn't applied — go back and re-check.

---

## Step 3 — Pick where your code will live (important)

**Rule:** clone into WSL's home directory, **NOT** into `C:\Users\Manoj\…`.

From inside Ubuntu (WSL), your home is `/home/<your-linux-username>`. Accessed from Windows, the same place is `\\wsl.localhost\Ubuntu\home\<your-linux-username>\`.

Open a WSL bash prompt (launch Ubuntu from Start, or type `wsl` in PowerShell) and make sure you're in your home directory:

```bash
cd ~
pwd
# should print: /home/<your-linux-username>
```

All further commands in this guide assume you're at this WSL bash prompt unless the step says otherwise.

---

## Step 4 — Install a few WSL tools you'll want

From WSL:

```bash
sudo apt update
sudo apt install -y git curl
```

(`sudo` will ask for the Linux password you set in Step 1.)

Set your Git identity — these should match the email on your Wikimedia developer account if you plan to push patches later:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## Step 5 — Clone MediaWiki core into WSL

Still in the WSL bash prompt, in `~`:

```bash
git clone https://gerrit.wikimedia.org/r/mediawiki/core.git mediawiki
cd mediawiki
```

This takes a few minutes — MediaWiki core is ~200 MB and has a long history. When it finishes:

```bash
pwd
# /home/<you>/mediawiki
ls
# You should see docker-compose.yml, DEVELOPERS.md, includes/, etc.
```

> **If you already plan to submit patches to Gerrit later**, you'll need to switch this remote from HTTPS to SSH. That's covered in the Gerrit guide — you can postpone it. HTTPS is enough for `git pull` and for now.

---

## Step 6 — Create the `.env` file

MediaWiki's Docker Compose setup reads environment variables from a `.env` file at the project root. `DEVELOPERS.md` specifies the contents exactly. From WSL, still in `~/mediawiki`:

```bash
cat > .env <<'EOF'
MW_SCRIPT_PATH=/w
MW_SERVER=http://localhost:8080
MW_DOCKER_PORT=8080
MEDIAWIKI_USER=Admin
MEDIAWIKI_PASSWORD=dockerpass
XDEBUG_CONFIG=
XDEBUG_ENABLE=true
XHPROF_ENABLE=true
EOF
```

Now the Windows-specific piece. `DEVELOPERS.md` says: *"Windows users: Run the following command to add a blank user ID and group ID to your .env file."* Because we're running through WSL, we're on Linux, but we still want to add `MW_DOCKER_UID` and `MW_DOCKER_GID` so file ownership works cleanly. Your WSL user has a UID/GID just like a real Linux user:

```bash
echo "MW_DOCKER_UID=$(id -u)" >> .env
echo "MW_DOCKER_GID=$(id -g)" >> .env
```

Verify the file looks right:

```bash
cat .env
```

You should see the 8 lines above plus two more (`MW_DOCKER_UID=1000`, `MW_DOCKER_GID=1000` typically — the exact numbers don't matter as long as they're there).

---

## Step 7 — Start the containers

From WSL, still in `~/mediawiki`:

```bash
docker compose up -d
```

First run, this will:

1. Pull `docker-registry.wikimedia.org/dev/bookworm-php83-fpm:…` (the PHP/FPM image, ~1 GB)
2. Pull a small Apache front-end image
3. Start both containers in the background (`-d`)

Takes 2–10 minutes depending on your connection. When it finishes, check they're running:

```bash
docker compose ps
```

Both services should show `running` or `Up`. If one shows `Exited`, run `docker compose logs` to see why.

---

## Step 8 — Install Composer dependencies inside the container

MediaWiki core needs its PHP libraries installed before the install script can run. This is done inside the `mediawiki` container:

```bash
docker compose exec mediawiki composer update
```

Takes a minute or two. You'll see a lot of "Installing …" output. A green "Generating autoload files" message near the end means success.

---

## Step 9 — Run the MediaWiki installer

This creates the SQLite database and the `LocalSettings.php` config file:

```bash
docker compose exec mediawiki /docker/install.sh
```

`install.sh` is a helper script bundled in the container that wraps the standard `maintenance/install.php` with all the right arguments drawn from your `.env` file. When it finishes, you should have a `LocalSettings.php` file in `~/mediawiki/` — verify:

```bash
ls LocalSettings.php
```

---

## Step 10 — Fix SQLite permissions (Windows/WSL only)

`DEVELOPERS.md` explicitly calls this out: *"The permissions with the cache/sqlite directory have to be set manually on Windows."*

```bash
docker compose exec mediawiki chmod -R o+rwx cache/sqlite
```

Skip this and you'll get "database is locked" errors on every page load.

---

## Step 11 — Verify it works

Open your browser (any Windows browser — Chrome, Firefox, Edge) and go to:

**http://localhost:8080/**

You should see the MediaWiki main page. The default admin login is:

- Username: `Admin`
- Password: `dockerpass`

If you got here, **your local MediaWiki is working.**

---

## Step 12 — Where to edit code from

You've got three equally valid options; pick whichever you prefer:

**Option A — VS Code with the WSL extension (recommended).** Install VS Code on Windows, then install its "WSL" extension. From WSL bash, run `code ~/mediawiki`. VS Code opens with the project, connected to WSL. File saves are instant, the integrated terminal is WSL bash, and everything Just Works.

**Option B — Windows File Explorer.** In File Explorer, type into the address bar: `\\wsl.localhost\Ubuntu\home\<your-linux-username>\mediawiki`. You can navigate, copy, drag-drop. Editing with a Windows editor like Notepad++ or Sublime also works.

**Option C — Editing from inside WSL with `vim` / `nano`.** Pure terminal, no GUI. Fine for quick tweaks.

> **Don't** copy the `mediawiki` folder out to `C:\` to edit it there — that reintroduces the slowness we eliminated in Step 3.

---

## Step 13 — Where to run commands from

Running commands that touch the repo or the containers needs to happen from *inside* the WSL `~/mediawiki` folder, because that's where `docker-compose.yml` and `.env` live.

Two ways to get there:

**From PowerShell** (per the tutorial snippet you quoted):
```powershell
cd \\wsl.localhost\Ubuntu\home\<your-linux-username>\mediawiki
```
You can then run `docker compose ...` commands from here. Docker Desktop is configured such that they'll be routed through WSL correctly.

**From WSL bash directly** (what I'd recommend — simpler, no path translation to think about):
- Open Ubuntu from the Start menu, or
- Open PowerShell and type `wsl` (drops you into bash inside Ubuntu), or
- Use VS Code's integrated terminal with the WSL extension.

Then:
```bash
cd ~/mediawiki
```

Inside WSL bash, the standard MediaWiki dev commands work without any path translation. For example:

```bash
# start containers
docker compose up -d

# tail logs
docker compose logs -f mediawiki

# open a shell inside the mediawiki container
docker compose exec mediawiki bash

# run a maintenance script
docker compose exec mediawiki php maintenance/run.php update

# run PHPUnit
docker compose exec mediawiki composer phpunit

# stop containers (preserves data)
docker compose down
```

Note the shell prompt changes when you `docker compose exec mediawiki bash` — you're now inside the container, at `/var/www/html/w`. Type `exit` to drop back to WSL.

---

## Step 14 — Daily-driver commands reference

| Task | Command (run from WSL `~/mediawiki`) |
|---|---|
| Start environment | `docker compose up -d` |
| Stop environment (keep data) | `docker compose down` |
| Stop + wipe containers and volumes | `docker compose down -v` |
| Follow logs | `docker compose logs -f mediawiki` |
| Open a bash shell in the container | `docker compose exec mediawiki bash` |
| Run a maintenance script | `docker compose exec mediawiki php maintenance/run.php <script> [args]` |
| Run DB schema updates (after installing an extension) | `docker compose exec mediawiki php maintenance/run.php update` |
| Run PHPUnit | `docker compose exec mediawiki composer phpunit` |
| Run linters | `docker compose exec mediawiki composer test` |
| Re-fix SQLite perms after a rebuild | `docker compose exec mediawiki chmod -R o+rwx cache/sqlite` |
| Fully reset the wiki (re-install from scratch) | Delete `LocalSettings.php` and `cache/sqlite/`, then re-run Step 9 & 10 |

---

## Step 15 — Installing extensions (e.g. Wikispeech)

Extensions go in `~/mediawiki/extensions/<Name>/`. From WSL:

```bash
cd ~/mediawiki/extensions
git clone https://gerrit.wikimedia.org/r/mediawiki/extensions/Wikispeech
cd ..
```

Enable it by appending to `LocalSettings.php`:

```bash
echo "wfLoadExtension( 'Wikispeech' );" >> LocalSettings.php
```

Run the DB schema updater:

```bash
docker compose exec mediawiki php maintenance/run.php update
```

Reload your browser at http://localhost:8080/ and go to `Special:Version` — the extension should appear in the list.

> **Wikispeech also needs Speechoid**, the TTS backend. That's a separate Docker Compose stack; see https://www.mediawiki.org/wiki/Extension:Wikispeech/Installing_Speechoid. Cover this after the base MediaWiki is working.

---

## Common problems and fixes

**"port 8080 already in use"** — Something else on Windows is using 8080 (another Docker project, a dev server, etc.). Either stop it, or change `MW_DOCKER_PORT=8080` to e.g. `8090` in `.env`, then `docker compose down && docker compose up -d`. Remember to update `MW_SERVER` in `.env` and access `http://localhost:8090/` in the browser.

**"permission denied" on cache/sqlite** — You forgot Step 10, or you just rebuilt the container. Re-run `docker compose exec mediawiki chmod -R o+rwx cache/sqlite`.

**Browser at http://localhost:8080 shows "Forbidden" or an empty page** — containers didn't start cleanly. Check `docker compose ps` and `docker compose logs mediawiki`.

**"/docker/install.sh: command not found" or similar** — Composer step (Step 8) didn't complete. Re-run `docker compose exec mediawiki composer update`.

**WSL is crushingly slow and you're sure code is inside WSL** — Docker Desktop's WSL 2 backend is disabled. Go back to Step 2, item 3.

**`docker: command not found` when you type it inside WSL bash** — WSL Integration is disabled. Go back to Step 2, item 4, and make sure Ubuntu is ticked there.

**"Cannot connect to the Docker daemon"** — Docker Desktop isn't running. Launch it from the Start menu, wait for the whale icon in the system tray to stop animating, then retry.

**After a Windows restart nothing works** — Docker Desktop needs to be launched manually (unless you set it to auto-start in its settings). WSL will wake up automatically when you open an Ubuntu terminal. Then `docker compose up -d` from `~/mediawiki`.

**Files created inside the container appear owned by weird UIDs in WSL** — your `MW_DOCKER_UID`/`MW_DOCKER_GID` in `.env` don't match your WSL user. Re-run the two `echo` lines in Step 6 and `docker compose down && docker compose up -d`.

---

## What to do next

1. Confirm MediaWiki works at http://localhost:8080/ and you can log in as Admin.
2. Read the official `DEVELOPERS.md` end-to-end (https://gerrit.wikimedia.org/g/mediawiki/core/+/HEAD/DEVELOPERS.md) — it has sections on Xdebug, XHProf, running tests, resetting the environment, and more, which are out of scope for this setup guide.
3. Clone Wikispeech into `extensions/` per Step 15 when you're ready for the actual contribution work.
4. When you're ready to push patches, follow the Gerrit guide (`wikimedia-gerrit-guide.md`) to set up SSH keys and `git-review` — note that those commands should be run from inside WSL, at `~/mediawiki` or `~/mediawiki/extensions/Wikispeech`, not from Windows PowerShell.
