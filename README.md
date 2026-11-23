<!--
Copyright (c) 2026 Robert August Vincent II <pillarsdotnet@gmail.com>
Co-author: Claude Code.
-->

# Claude Code, packaged for Ubuntu

[Claude Code](https://code.claude.com/docs) is Anthropic's agentic coding
tool for the terminal. Anthropic ships it as a `curl | bash` installer that
puts a private copy under each user's home directory and updates it behind
the system's back, which suits one workstation and not a fleet. This
repository turns it into a package in the
[registry](https://forgejo.bobbalisa.org/pillarsdotnet/-/packages) on
`forgejo.bobbalisa.org`, so the fleet gets one system-wide copy whose
version `apt` decides.

## The package ships none of Anthropic's code

Claude Code is proprietary: the `npm` metadata records its licence as
`SEE LICENSE IN README.md`, and that README names the Anthropic Commercial
Terms, which grant no right to redistribute the program. The registry on
`forgejo.bobbalisa.org` is readable without credentials, so a package
carrying the binary would be republishing it.

So the package carries the digests and nothing else, and its `postinst`
downloads the release from Anthropic's own service onto the machine being
installed, exactly as the upstream installer would:

|                                          |                                     |
| ---------------------------------------- | ----------------------------------- |
| `/usr/share/claude-code/version`         | the release to fetch                |
| `/usr/share/claude-code/upstream-sha256` | its `SHA-256`, per platform         |
| `/usr/lib/claude-code/claude`            | a stub, overwritten by the download |
| `/usr/bin/claude`                        | symlink to the above                |

The built package is about 5 KB. The compressed download is 104 MB against
294 MB for the bare binary, so the `postinst` fetches `claude.zst`,
truncates it to the length the manifest records -- the upstream installer
does the same, so the stream cannot be assumed to end where the binary does
-- and checks the result against the digest of the binary. It falls back to
the uncompressed download if any of that fails, and refuses to install
anything whose digest does not match.

Whoever installs the package receives Claude Code directly from Anthropic
and under Anthropic's terms. See [`debian/copyright`](debian/copyright).

## Why the stub, rather than just downloading to the path

Claude Code detects a package-managed installation by running `dpkg -S` on
the executable it is running from, on any system whose `/etc/os-release`
names `debian` in `ID` or `ID_LIKE`. When `dpkg` owns that path, Claude
Code **turns its own updater off** and `claude update` says so:

```console
$ claude update
Current version: 2.1.318
Checking for updates to latest version...

Claude is managed by a package manager.
Please use your package manager to update.
```

A binary that `postinst` created at a path the package does not list is not
owned by `dpkg`, and Claude Code would then treat a root-owned system
install as a private one and try to update it in place, behind `apt`. So
the package ships a stub at that exact path and the `postinst` writes over
it: `dpkg` answers ownership questions from its own file list rather than
from the file, so overwriting does not change the answer.

Two consequences worth knowing. `dpkg -V` and `debsums` report
`/usr/lib/claude-code/claude` as modified, because it is. And every package
upgrade re-downloads, because `dpkg` unpacks the stub over the binary
before `postinst` runs -- which is the right thing when the point of the
upgrade is a new release, and costs nothing when `postinst` finds the
binary already matching the digest and returns without fetching.

## Staying current

`.forgejo/workflows/upstream.yml` checks Anthropic's `stable` channel daily,
and when it names a release the tree does not, commits the two files that
say otherwise and pushes to `main`. That push is what builds and uploads the
package, through the Registry workflow. Releases come several times a week,
so the registry stays at most a day behind `stable`; run the Upstream
workflow by hand to pick one up sooner, or give it a version to pin to an
older release or jump ahead of `stable`.

Doing it by hand is the same one commit:

```sh
./refresh-upstream          # or: ./refresh-upstream 2.1.318
git commit -m "Install the upstream $(dpkg-parsechangelog -SVersion | sed 's/-[^-]*$//') release." \
  -- debian/changelog debian/upstream-sha256
```

`refresh-upstream` rewrites `debian/changelog` instead of appending to it,
unlike the other repacking packages here. The changelog holds one entry
because the package holds no code of its own: the entry says which release
the `postinst` fetches, and git history is the log of how that changed.
`dch` merges a new version into an existing `UNRELEASED` entry rather than
replacing it, so bumping through `dch` would accumulate a bullet per
release forever.

The series suffix (`~ppa1~ubuntu<release>.1`) is added at build time by
[`.forgejo/registry-upload`](.forgejo/registry-upload) and never committed.

### Getting it onto the fleet

`apt` only installs what something asks it to. `unattended-upgrades` runs
daily on the fleet but its `Allowed-Origins` lists only Ubuntu's own
origins, so the registry is not among them. To have hosts follow the
registry without being asked:

```conf
// /etc/apt/apt.conf.d/51unattended-upgrades-forgejo
Unattended-Upgrade::Allowed-Origins {
        "Forgejo on homelab:${distro_codename}";
};
```

`Forgejo on homelab` is the `Origin` the registry's `Release` file declares,
and the suite is the series codename. That file belongs in the `sources`
role of `kubuntu_bootstrap_ansible`, next to the repository entry it
matches; note that it opts the fleet into unattended upgrades of **every**
package in the registry, not just this one.

## Architectures

The package is `Architecture: all` and the `postinst` chooses the download
from `dpkg --print-architecture`: `amd64` takes `linux-x64` and `arm64`
takes `linux-arm64`. One build therefore covers both, which also means an
`arm64` host can install it from a registry that indexes only `all` and
`amd64`, as this one does. Anthropic also publishes `musl` builds; Ubuntu
has no use for them and they are not recorded.

The binary needs nothing but `glibc`: it is dynamically linked against
`libc`, `libm`, `librt`, `libpthread` and `libdl`, and its highest versioned
symbol is `GLIBC_2.26`, so it runs on every series this targets.

## Licence

The packaging is published under the Apache License 2.0. Claude Code itself
is not; see [`debian/copyright`](debian/copyright).
