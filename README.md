# sloth-yocto

Top-level workspace for the **Sloth** Yocto build (thermal-camera deer
detection ECU, Jetson Orin Nano Super devkit).

From a fresh machine to a built image takes five steps:

1. [Clone this repo with its submodules](#1-clone-with-submodules)
2. [Build the Docker image](#2-build-the-docker-image)
3. [Prepare the caches and your SSH key](#3-prepare-caches-and-ssh-key)
4. [Enter the container](#4-enter-the-container)
5. [Build with kas](#5-build-with-kas)

## What is in this repo

```
~/yocto/                       ← this repo (sloth-yocto)
├── meta-sloth/                ← submodule: product layer + kas configs
├── yocto-jetson-buildenv/     ← submodule: Dockerfile, yocto-shell.sh, site.conf
├── docker -> yocto-jetson-buildenv/docker
│
│   created by the steps below, never committed:
├── sloth-authorized_keys      ← your SSH public key, baked into the image
├── kas-fetch/                 ← upstream layers cloned by kas (bitbake, oe-core, meta-tegra, ...)
│   └── build/                 ← bitbake build directory
└── flash/                     ← unpacked tegraflash bundles
```

Repos we commit to are **submodules**. Upstream layers are **not** in git here:
kas clones them at the commits pinned in
[`meta-sloth/kas/sloth-orin-nano-fetch.yml`](meta-sloth/kas/sloth-orin-nano-fetch.yml).

## Requirements

- Linux or WSL2 (Ubuntu) with **Docker** working: `docker run --rm hello-world`
  must succeed without `sudo`.
- ~250 GB free disk for the build tree, plus the shared caches.
- On WSL2, keep everything under `~` and `/srv`, never `/mnt/c`.

## 1. Clone with submodules

```bash
git clone --recursive https://github.com/RenMurakami/sloth-yocto.git ~/yocto
cd ~/yocto
```

Already cloned without `--recursive`? `meta-sloth/` and
`yocto-jetson-buildenv/` will be empty. Fill them in with:

```bash
git submodule update --init
```

Check: both lines should start with a space (not `-` or `+`):

```bash
git submodule status
#  a06d142... meta-sloth (heads/main)
#  9e29672... yocto-jetson-buildenv (heads/main)
```

## 2. Build the Docker image

```bash
cd ~/yocto
docker build -t yocto-jetson:wrynose docker/
```

This takes a few minutes the first time. Rebuild it whenever
`docker/Dockerfile` or `docker/site.conf` changes (for example after a
`git submodule update` that moves `yocto-jetson-buildenv`).

Check: `docker images yocto-jetson` lists the `wrynose` tag.

## 3. Prepare caches and SSH key

Once per machine:

```bash
# Shared download and sstate caches, mounted into the container
sudo mkdir -p /srv/yocto/{downloads,sstate-cache}
sudo chown -R "$USER" /srv/yocto

# Public key that will be allowed to ssh in as root on the device
cp ~/.ssh/id_ed25519.pub ~/yocto/sloth-authorized_keys
```

## 4. Enter the container

Always run it **from `~/yocto`**: that directory is mounted as `/work`.

```bash
cd ~/yocto
./docker/yocto-shell.sh
```

You get a shell inside the container as your own user, in `/work`
(= `~/yocto` on the host). Files you create there appear on the host too.
`exit` leaves; the container is removed, but `/work` and `/srv/yocto` are kept.

To run one command instead of a shell: `./docker/yocto-shell.sh <command...>`.

## 5. Build with kas

Inside the container:

```bash
cd /work
KAS_WORK_DIR=/work/kas-fetch \
SLOTH_SSH_AUTHORIZED_KEYS=/work/sloth-authorized_keys \
    kas build meta-sloth/kas/sloth-orin-nano-fetch.yml
```

- The first run clones the upstream layers into `kas-fetch/`; later runs reuse
  them without network access.
- The image ends up in
  `kas-fetch/build/tmp/deploy/images/sloth-orin-nano-sd/`.
  Set `KAS_BUILD_DIR` to put the build directory elsewhere.
- For a bitbake shell (`bitbake -c ...`, `devtool`), use `kas shell` with the
  same arguments instead of `kas build`.

## Day-to-day git

**Pull everything, including new submodule commits:**

```bash
git pull --recurse-submodules
git submodule update --init
```

**Change meta-sloth (or yocto-jetson-buildenv):** submodules are checked out
at a fixed commit (detached HEAD), so switch to the branch first.

```bash
cd meta-sloth
git switch main
# edit, then
git commit -am "..." && git push

cd ..
git add meta-sloth                       # record the new commit here
git commit -m "Bump meta-sloth" && git push
```

**Update upstream layers:** edit the commits in
`meta-sloth/kas/sloth-orin-nano-fetch.yml`. Update all of them together, taken
from the submodule pins of one tegra-demo-distro wrynose commit. That is a
meta-sloth change, then bump the submodule here as above.
