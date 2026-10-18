# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bundle install
bin/kitchen list                                 # 3 instances, one suite
bin/kitchen test default-debian-13-3008-2        # single instance
bin/kitchen verify default-debian-11-3006-27     # re-run InSpec only
pre-commit run --all-files                       # all lint hooks, as the CI does
```

Instances: `default-debian-13-3008-2`, `default-debian-13-3006-27`,
`default-debian-11-3006-27`.

## Architecture — how this fork departs from the template

- **One state file.** All logic lives in `timezone/init.sls`; there is no
  `package/` / `config/` / `service/` split, no `libtofs.jinja`, no managed
  templates. The only states are `timezone` and the test-only
  `timezone._mapdata`.
- **Ordering in `init.sls` is the point of the formula.** Two IDs feed
  `timezone_setting` (`timezone.system`) through `require_in`:
  - `timezone_packages` installs `tzdata`, preceded by a `file.symlink`
    of `/etc/localtime` to `Etc/UTC` that only renders when `/etc/localtime`
    is missing at render time (unattended tzdata install needs it, and so does
    Salt's current-zone comparison);
  - `dbus_for_timezone` installs and runs `dbus`, because Salt's timezone
    module drives `timedatectl`, which needs D-Bus.
  - The package/dbus block is skipped on `MacOS` and `Windows`.
- **Map files are no-ops on Debian.** `defaults.yaml` holds every value used
  on Debian; `osfamilymap.yaml`, `osmap.yaml`, `osfingermap.yaml` and
  `osarchmap.yaml` carry only non-Debian overrides (Suse, Gentoo, FreeBSD,
  OpenBSD) or empty dicts.
- **Pillar keys**: `timezone:name`, `timezone:utc`, plus `pkg:name`,
  `dbus:{pkg,service,run_service}`, `path_localtime`, `path_zoneinfo`,
  `tz_UTC` (overridable directly or under `timezone:lookup`). `defaults.yaml`
  defaults `name` to `Europe/Berlin`.

## Tests

- One suite, `default`, with pillar `test/salt/pillar/timezone.sls`
  (`Europe/Paris`, `utc: true`).
- Controls: `package_spec.rb` (tzdata installed) and `config_spec.rb`
  (`/etc/timezone` contains `Europe/Paris` on Debian, `/etc/localtime` symlink
  elsewhere). The literal `Europe/Paris` in the control must follow the test
  pillar.
- `timezone._mapdata` runs in `state_top`, but this fork has **no**
  `test/integration/default/files/_mapdata/` reference dumps and no control
  comparing them: the merged map is dumped, not asserted.
