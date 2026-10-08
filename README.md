# vykar.pservice

> ⚠️ This repository, including the documentation below, is entirely AI slop and barely tested. USE AT YOUR OWN RISK. ⚠️

[mkosi](https://github.com/systemd/mkosi) configuration that packages the
upstream [vykar](https://github.com/borgbase/vykar) release binary as a
systemd service, in either of two formats:

| Format   | Command                    | Output                             |
|----------|----------------------------|------------------------------------|
| sysext   | `mkosi -t sysext build`    | `mkosi.output/sysext/vykar.raw`    |
| portable | `mkosi -t portable build`  | `mkosi.output/portable/vykar.raw`  |

Both provide `vykar.service` for the system manager and for user managers,
running `vykar daemon`. Each instance only starts if its configuration exists:

- system: `/etc/vykar/config.yaml`
- user: `$XDG_CONFIG_HOME/vykar/config.yaml` (normally `~/.config/vykar/config.yaml`)

For encrypted repositories the daemon needs a non-interactive passphrase source;
see the [vykar daemon docs](https://vykar.borgbase.com/daemon.html).

## Releases

CI builds both formats for x86-64 and arm64 on every push and pull request.
Publishing a GitHub release attaches the images to it, named per
[systemd.v(7)](https://www.freedesktop.org/software/systemd/man/latest/systemd.v.html),
along with a `SHA256SUMS` manifest for `systemd-sysupdate`:

- `vykar_<version>_<arch>.sysext.raw`
- `vykar_<version>_<arch>.raw` (portable)

Tag releases `<vykar version>-<N>`, starting at `-0` for each new vykar version
and counting up for packaging-only changes. Pre-releases use a lettered suffix
such as `-alpha.1` or `-rc.0`, published as GitHub pre-releases. The tag becomes
the image version, and these sort as
`0.20.1-alpha.1 < 0.20.1-rc.0 < 0.20.1-0 < 0.20.1-1 < 0.20.2-0`, so sysupdate
sees each release as newer. CI rejects tags that don't start with
`<mkosi.version>-`.

A bare `<vykar version>` sorts below all of its suffixed versions, so don't use
it for releases.

Each image has a GitHub build provenance attestation:

```sh
gh attestation verify vykar_0.20.1-0_x86-64.sysext.raw --repo <owner>/vykar.pservice
```

The examples below install these into `.v/` directories. systemd then uses the
newest version for the local architecture, so updating just means adding a file.
For local builds, copy `mkosi.output/<format>/vykar.raw` in under the same naming
scheme.

## sysext

The extension adds `/usr/bin/vykar` and the units to the host's `/usr`, so vykar
runs as an ordinary host service with full access to the host: all files, host
tools for `hooks`/`command_dumps`/`passcommand`, `~/.ssh`, and so on.

```sh
sudo install -Dm0644 -t /var/lib/extensions/vykar.sysext.raw.v/ vykar_0.20.1-0_x86-64.sysext.raw
sudo systemd-sysext refresh
```

The units are enabled by the extension itself, so nothing else is needed:
adding a config file and rebooting (or `systemctl [--user] start vykar`) is
enough. To opt a host or user out despite a config file, use
`systemctl [--user] mask vykar`.

The extension is marked `ID=_any`, so it stays applied across OS and bootc
image upgrades.

After adding a new version, restart running instances yourself; see
`extension-release.vykar` for why `EXTENSION_RESTART_UNITS=` is not used:

```sh
sudo systemd-sysext refresh
sudo systemctl try-restart vykar
systemctl --user try-restart vykar
```

### Updates with systemd-sysupdate

The extension ships its own transfer definition as the `vykar` component
(`/usr/lib/sysupdate.vykar.d/vykar.transfer`). It downloads the newest image
from this repository's latest GitHub release into the `.v/` directory above and
keeps the previous version around:

```sh
sudo systemd-sysupdate --component=vykar list
sudo systemd-sysupdate --component=vykar update
```

Only the download is automated. The new version is used after the next
`systemd-sysext refresh`, which normally means the next reboot.

The stock `systemd-sysupdate-update.timer` covers all components only from
systemd 262 (`update --component-all`). Older versions update just the
component-less `sysupdate.d/` set, so run the command above yourself.

GitHub releases have no `SHA256SUMS.gpg`, so the transfer uses `Verify=no`.
Downloads are still checked against `SHA256SUMS`, fetched over HTTPS, but
sysupdate does not check the GitHub attestations.

## Portable service

The image contains only the static binary, so the service sees the host only
through the bind mounts in its units: host `/etc` (read-only) and `/var`, plus
`/home` for user instances. On bootc systems that covers everything worth
backing up, including user homes in `/var/home`.

Attach with the `trusted` profile. The `default` profile's `PrivateUsers=`
hides the ownership of other users' files, so a system backup can't read them.

```sh
sudo install -Dm0644 -t /var/lib/portables/vykar.raw.v/ vykar_0.20.1-0_x86-64.raw
sudo portablectl attach --profile=trusted --enable --now /var/lib/portables/vykar.raw.v
```

The `.v/` path is resolved when the image is attached, so after adding a new
version run `sudo portablectl reattach --now /var/lib/portables/vykar.raw.v`.
For a user instance, keep the `.v/` directory somewhere you own (for example
`~/.local/share/portables/vykar.raw.v/`) and use `portablectl --user` with its
absolute path.

The image can't carry a transfer definition for the host, so to download
updates with `systemd-sysupdate --component=vykar-portable update`, add this as
`/etc/sysupdate.vykar-portable.d/vykar.transfer` (see the sysext section for
`Verify=no` and timer caveats):

```ini
[Transfer]
Verify=no

[Source]
Type=url-file
Path=https://github.com/blue42u/vykar.pservice/releases/latest/download
MatchPattern=vykar_@v_%a.raw

[Target]
Type=regular-file
Path=/var/lib/portables/vykar.raw.v
MatchPattern=vykar_@v_%a.raw
Mode=0444
InstancesMax=2
```

Limitations compared to the sysext:

- There is no shell or other tools in the image, so `hooks`, `command_dumps`
  and `passcommand` can't work. Use `passphrase` or `VYKAR_PASSPHRASE` instead.
- No `ExecReload=`; reload the config with `systemctl [--user] kill -s HUP vykar`.
- User-scope portable services need a recent systemd with `systemd-portabled`,
  `systemd-mountfsd` and `systemd-nsresourced` available to users.

## Updating vykar

Bump `mkosi.version` and replace `SHA256SUMS` with the checksums of the new
`*-unknown-linux-musl.tar.gz` release assets (shown as `digest` on the GitHub
release assets). `mkosi.sync` downloads the tarball into `downloads/` and fails
if the checksum for the architecture being built is missing or doesn't match.
Then publish a release tagged `<new version>-0`.
