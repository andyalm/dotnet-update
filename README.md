# dotnet-update

A small script for managing .NET on macOS the way servicing actually works:

- **One SDK**, always the latest release. A current SDK can build projects
  targeting older versions of .NET, so there's rarely a reason to keep more
  than one (older SDKs are pruned automatically).
- **Multiple ASP.NET Core runtime majors**, each kept at its latest patch.
  Which majors you keep is *your* explicit choice — a plain `dotnet-update`
  only patches what's already installed, and you `add`/`remove` majors when
  you decide to, not when a support window says so.

## Why not Homebrew + dotnet-install.sh?

Homebrew works fine for keeping a single SDK current, but Microsoft does not
publish a macOS installer for the ASP.NET Core *runtime* — it's only available
via `dotnet-install.sh`. And the two can't be combined: since .NET 7, the
`dotnet` host only resolves SDKs and runtimes located under its own root
(`DOTNET_ROOT`), and Homebrew installs into a versioned Cellar path that is
replaced on every upgrade, orphaning anything you side-load there.

So this script skips Homebrew entirely and manages everything under a single
`~/.dotnet` root with `dotnet-install.sh`, using Microsoft's
[release metadata feed](https://github.com/dotnet/core/blob/main/release-notes/releases-index.json)
to find the latest SDK.

## Requirements

- macOS (Linux should work too, but is untested)
- `curl` and `jq` (both ship with recent macOS)
- No existing .NET install in another location — in particular, uninstall any
  Homebrew `dotnet` first (`brew uninstall dotnet`), since two roots can't see
  each other's runtimes.

## Install

Clone the repo and symlink the script somewhere on your `PATH`:

```sh
git clone https://github.com/andyalm/dotnet-update.git
mkdir -p ~/.local/bin
ln -s "$PWD/dotnet-update/dotnet-update" ~/.local/bin/dotnet-update
```

Then add the .NET root to your shell profile (e.g. `~/.zshrc`), including
`~/.local/bin` if it isn't already on your `PATH`:

```sh
export PATH="$HOME/.local/bin:$PATH"

export DOTNET_ROOT="$HOME/.dotnet"
export PATH="$DOTNET_ROOT:$DOTNET_ROOT/tools:$PATH"
```

(`$DOTNET_ROOT/tools` is where `dotnet tool install -g` puts global tools.)

Open a new shell, then run the initial install — the latest SDK plus whichever
runtime majors you want to track:

```sh
dotnet-update
dotnet-update add 9.0
dotnet-update add 8.0
```

## Usage

```sh
dotnet-update             # update SDK to latest release and update each
                          # installed runtime major to its latest patch;
                          # prune superseded versions
dotnet-update add 8.0     # start tracking a runtime major
dotnet-update remove 8.0  # purge a runtime major
```

Run `dotnet-update` whenever you want to pick up servicing releases — .NET
patches normally ship on the second Tuesday of each month. It's idempotent and
only downloads what's missing.

Things to know:

- The SDK bundles the runtime of its own major, so when a new .NET version is
  released, updating the SDK automatically starts tracking that runtime major.
  Older majors stick around until you `remove` them.
- `remove` refuses to purge the major the current SDK depends on.
- Everything lives under `~/.dotnet`; to uninstall .NET completely, just
  delete that directory.
