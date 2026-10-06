# secure-containerisation

Hardened, digest-pinned container images for AI coding agents.

## Quick start

Images are published to the GitHub Container Registry (GHCR) and tagged **only by commit SHA**. There is no `latest` tag. Pick a SHA from this repo's [GHCR packages page](https://github.com/Jubblin?tab=packages&repo_name=secure-containerisation), then pull and run it with the hardening flags:

```sh
SHA=<git-sha-from-packages-page>
docker pull ghcr.io/jubblin/secure-containerisation/termic:$SHA

docker run --rm -it \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --read-only \
  --tmpfs /tmp \
  ghcr.io/jubblin/secure-containerisation/termic:$SHA
```

What the flags do:

- `--cap-drop=ALL` removes all Linux capabilities (the kernel privileges root normally has).
- `--security-opt=no-new-privileges:true` stops processes gaining privileges, for example through setuid binaries.
- `--read-only` mounts the container's root filesystem read-only.
- `--tmpfs /tmp` gives the container a writable, in-memory `/tmp`.

With `--read-only`, agents cannot save config or work files. Mount a volume for any path they need to write, for example `-v "$PWD":/workspace`.

To build locally instead of pulling:

```sh
docker build -t termic -f builds/termic/Containerfile builds/termic
```

Then run `termic` with the same flags.

## What this repo is

Each image installs one or more AI coding agents on a base image pinned by `sha256` digest, so the same build input always gives the same base. CI builds every image for amd64 and arm64 and publishes it to GHCR. Renovate keeps the pinned digests up to date.

## Layout

| Path | Purpose |
| --- | --- |
| `builds/<name>/Containerfile` | An image that CI builds and publishes automatically. |
| `samples/Containerfile` | A reference hardened pattern. CI does not build it. |
| `.github/workflows/build-containers.yml` | The build and publish workflow. |
| `renovate.json` | Renovate dependency-update config. |

`samples/Containerfile` shows the hardening pattern to copy:

- runs as a non-root user (`hermes`, uid 10001);
- pins every apt package to an exact version;
- downloads the installer script and checks its `sha256` before running it.

## Images

| Image | Base | Contents |
| --- | --- | --- |
| `hermes-agent` | `nousresearch/hermes-agent` | Nous Research hermes-agent, plus Bitwarden CLI (`@bitwarden/cli`) and Claude Code. Runs as user `hermes`. |
| `termic` | `node` (current LTS, Debian) | Sandbox with `git`, `ripgrep`, `gh`, `glab`, and the agents Claude Code, Codex, Copilot, opencode, grok, antigravity, pi and muse. Working directory `/workspace`. No `USER` is set, so plain `docker run` runs as root; termic runs it as your uid (`--user "$(id -u):$(id -g)"`), and `/root` (`HOME`, where agents keep config) is world-writable for that. |

In `termic` the agents are not version-pinned; each build installs the latest release. `glab` and muse are optional: if their install fails, the build still succeeds without them.

## CI and tags

The workflow [`build-containers.yml`](.github/workflows/build-containers.yml):

1. Finds every `builds/*/` directory that contains a `Containerfile`.
2. Builds each one for `linux/amd64` and `linux/arm64`.
3. On push and manual runs, pushes `ghcr.io/jubblin/secure-containerisation/<dir>:<git-sha>`.
4. On pull requests to `main`, builds only and pushes nothing.

## Building locally

```sh
docker build -t <name> -f builds/<name>/Containerfile builds/<name>
```

`termic` takes a build argument to re-fetch its unpinned agents without re-running the slower base and apt layers. Change the value to force a refresh:

```sh
docker build -t termic --build-arg TERMIC_AGENT_REFRESH=$(date +%s) \
  -f builds/termic/Containerfile builds/termic
```

## Runtime hardening

Some protections cannot be set in a Containerfile. Apply them at `docker run`, or in your orchestrator's pod or container security context:

```sh
--cap-drop=ALL --security-opt=no-new-privileges:true --read-only --tmpfs /tmp
```

## Adding an image

1. Create `builds/<name>/Containerfile`. CI picks it up with no workflow change.
2. Pin the base image by digest: `FROM image@sha256:<digest>`.
3. Follow `samples/Containerfile` where you can: non-root user, pinned packages, verified installers.

## Dependency updates

[Renovate](https://docs.renovatebot.com/) opens pull requests that bump the pinned base-image digests. Its config is in `renovate.json`.
