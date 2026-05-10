# eigenwallet-images

Builds container images for the [eigenwallet/core](https://github.com/eigenwallet/core) atomic-swap stack and publishes them to GHCR. Used by the `apps/eigenwallet/` deployment in `homelab/`.

## What gets built

For each release, four images:

| Image | Source repo | Path | Ref |
|---|---|---|---|
| `ghcr.io/tylerjw/eigenwallet-asb` | `eigenwallet/core` (upstream) | `swap-asb/Dockerfile` | `$version` |
| `ghcr.io/tylerjw/eigenwallet-asb-controller` | `eigenwallet/core` (upstream) | `swap-controller/Dockerfile` | `$version` |
| `ghcr.io/tylerjw/eigenwallet-rendezvous-node` | `eigenwallet/core` (upstream) | `libp2p-rendezvous-node/Dockerfile` | `$version` |
| `ghcr.io/tylerjw/eigenwallet-swap-cli` | `tylerjw/eigen-patches` (fork) | `Dockerfile.swap-cli` | `patches/$version-headless-taker` |

The fork re-adds `list-sellers`, `deposit-address`, and `buy-xmr` subcommands that upstream removed in 4.4.x. Used by `market-observer/` competitor surveys and the `recycle-btc` BTC→XMR replenishment workflow.

Linux/amd64 only — k8s-w-01 is x86.

## How to release a new version

Two ways:

**1. Bookmark-driven (preferred):** push a `release/<version>` bookmark.

```sh
jj new -m "build eigenwallet 4.5.1"
jj bookmark create release/4.5.1 -r @
jj git push --bookmark release/4.5.1
```

The workflow extracts `4.5.1` from the branch name. Upstream images build from `eigenwallet/core@4.5.1`; `swap-cli` builds from `tylerjw/eigen-patches@patches/4.5.1-headless-taker`. All four images are tagged `4.5.1` and `latest`.

(We use a branch instead of a git tag because `jj git push` doesn't push tags — bookmarks are jj's native unit. The branch is harmless once pushed; you can leave it or delete it.)

**2. Manual dispatch:** Actions tab → `build` → `Run workflow` → set `eigenwallet_ref` (e.g. `main`, `4.5.0`, a sha). Useful for ad-hoc rebuilds.

## Pinning in homelab

Each job prints a digest at the end. Pin it in `apps/eigenwallet/*.yaml` to make rollouts deterministic:

```yaml
image: ghcr.io/tylerjw/eigenwallet-asb:4.5.0@sha256:...
```

Don't pin to `:latest` from k8s — it's there for convenience, not deployment.

## Notes

- Builds Rust from source, so first run is slow (~30 min on free runners). GHA layer cache helps subsequent runs hit Cargo's cached dep layers.
- Images need to be made public on GHCR (or the homelab needs an `imagePullSecret`) before k8s can pull them. The `tarot` namespace already uses an `imagePullSecret` for ghcr — same pattern works here.
- For the `swap-cli` build to work, `tylerjw/eigen-patches` must be public (or you have to swap to a PAT-based clone in the workflow). `secrets.GITHUB_TOKEN` only authenticates against the calling repo.
- Fork branch convention: `patches/<upstream-version>-headless-taker`, rebased onto upstream tags. If you ever rename the branch suffix, update the workflow's ref-resolution step.
