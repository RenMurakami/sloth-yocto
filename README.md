# sloth-yocto

Top-level workspace for the **Sloth** Yocto build (thermal-camera deer
detection ECU, Jetson Orin Nano Super devkit). It ties together the two repos
we own; every upstream layer is fetched by kas at a pinned commit.

| Path | What | Managed by |
|---|---|---|
| `meta-sloth/` | Product layer: distro, machine, images, recipes, kas configs | git submodule |
| `yocto-jetson-buildenv/` | Docker build environment (`yocto-shell.sh`, `site.conf`) | git submodule |
| `docker` | Symlink to `yocto-jetson-buildenv/docker` | this repo |
| `kas-fetch/` | bitbake, openembedded-core, meta-openembedded, meta-tegra, meta-tegra-community, meta-virtualization, tegra-demo-distro | kas (`meta-sloth/kas/sloth-orin-nano-fetch.yml`), not committed |
| `build*/`, `flash/` | Build output and flash packages | not committed |

Rule of thumb: repos we commit to are submodules; upstream layers are pinned in
the kas file. A repo is never pinned in both places.

## Setup

```bash
git clone --recursive https://github.com/RenMurakami/sloth-yocto.git ~/yocto
cd ~/yocto
docker build -t yocto-jetson:wrynose docker/
sudo mkdir -p /srv/yocto/{downloads,sstate-cache} && sudo chown -R "$USER" /srv/yocto
cp ~/.ssh/id_ed25519.pub sloth-authorized_keys   # key baked into the image
```

If you cloned without `--recursive`, run `git submodule update --init`.

## Build

```bash
./docker/yocto-shell.sh          # enters the container, cwd = /work (this directory)

# inside the container
KAS_WORK_DIR=/work/kas-fetch \
SLOTH_SSH_AUTHORIZED_KEYS=/work/sloth-authorized_keys \
    kas build meta-sloth/kas/sloth-orin-nano-fetch.yml
```

The first run clones the upstream layers into `kas-fetch/`; later runs reuse
them without network access.

## Updating

- **meta-sloth / yocto-jetson-buildenv:** commit and push inside the submodule,
  then commit the new submodule pointer here (`git add meta-sloth`).
- **Upstream layers:** update all commits in `sloth-orin-nano-fetch.yml`
  together, taken from the submodule pins of one tegra-demo-distro wrynose
  commit. That change belongs to meta-sloth.
