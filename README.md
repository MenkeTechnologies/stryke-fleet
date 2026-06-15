```
 ███████╗██╗     ███████╗███████╗████████╗
 ██╔════╝██║     ██╔════╝██╔════╝╚══██╔══╝
 █████╗  ██║     █████╗  █████╗     ██║
 ██╔══╝  ██║     ██╔══╝  ██╔══╝     ██║
 ██║     ███████╗███████╗███████╗   ██║
 ╚═╝     ╚══════╝╚══════╝╚══════╝   ╚═╝
        [ p a r a l l e l   e x p e c t ]
```

[![CI](https://github.com/MenkeTechnologies/stryke-fleet/actions/workflows/ci.yml/badge.svg)](https://github.com/MenkeTechnologies/stryke-fleet/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![stryke](https://img.shields.io/badge/stryke-package-cyan.svg)](https://github.com/MenkeTechnologies/strykelang)

### `[EXPECT, BUT N SESSIONS AT ONCE]`

> *"Tcl/Expect automated one terminal in 1990. This automates fifty, in parallel, from a playbook."*

`stryke-fleet` is parallel expect/PTY automation as a pure-stryke package: transcripted sessions, declarative playbooks, a recipe corpus for the interactive CLIs everyone scripts by hand (ssh, sftp, telnet, sudo, su, passwd, psql, mysql, redis-cli, docker login, installers, network gear), and multi-host fan-out on stryke's thread pool. The PTY primitives (`pty_spawn`, `pty_expect`, `pty_expect_table`, …) and the parallelism (`pmap`) are stryke builtins — this package is the orchestration layer. No `[ffi]` table, no cdylib, no helper binary — just `.stk` modules loaded on `use Fleet`. Created by MenkeTechnologies.

### [`strykelang`](https://github.com/MenkeTechnologies/strykelang) &middot; [`MenkeTechnologiesMeta`](https://github.com/MenkeTechnologies/MenkeTechnologiesMeta) · [`stryke-utils`](https://github.com/MenkeTechnologies/stryke-utils) · [`stryke-mcpd`](https://github.com/MenkeTechnologies/stryke-mcpd)

---

## Table of Contents

- [\[0x00\] Why a Package, Not Core](#0x00-why-a-package-not-core)
- [\[0x01\] Install](#0x01-install)
- [\[0x02\] Quick Start](#0x02-quick-start)
- [\[0x03\] Sublibraries](#0x03-sublibraries)
- [\[0x04\] What's NOT in Here](#0x04-whats-not-in-here)
- [\[0x05\] CLI](#0x05-cli)
- [\[0x06\] Tests](#0x06-tests)
- [\[0x07\] Layout](#0x07-layout)
- [\[0xFF\] License](#0xff-license)

---

## [0x00] Why a Package, Not Core

The spawn/expect/send loop is already core: `pty_spawn`, `pty_send`, `pty_read`, `pty_expect`, `pty_expect_table`, `pty_buffer`, `pty_alive`, `pty_eof`, `pty_close`, `pty_interact` are stryke builtins, and `pmap` gives every stryke program a thread pool for free. What core deliberately does NOT ship is opinion: what a "step" is, what a failed run returns, what an ssh login chain looks like, how a transcript is shaped. That's policy, and policy belongs in a package where it can version independently:

- **`Fleet::Session`** — transcripted sessions. Every send, match, timeout, and close is recorded, so a failed run hands you the full interaction log instead of a stuck process.
- **`Fleet::Playbook`** — declarative step lists. `+{expect, send, timeout, branches, optional}` hashrefs run in order; first non-optional timeout fails the run with the step name.
- **`Fleet::Recipes`** — the corpus. Login chains and prompt dances as pure data generators — composable, splice-able, unit-testable without a target host.
- **`Fleet::Fanout`** — the part Tcl/Expect never had. One playbook, N targets, one PTY per thread, results in target order, wall-clock of the slowest host.

## [0x01] Install

```sh
# From a release:
s pkg install -g github.com/MenkeTechnologies/stryke-fleet

# From a local checkout:
git clone https://github.com/MenkeTechnologies/stryke-fleet
cd stryke-fleet
s pkg install -g .              # installs into ~/.stryke/store/stryke-fleet@<version>/

# Or via Makefile:
make install
```

No cargo step. No cdylib. The installed store directory contains only `stryke.toml` + `lib/*.stk` — no compiled artifacts, fully portable across platforms.

## [0x02] Quick Start

```perl
use Fleet

# One session, transcripted
val $s = Fleet::Session::open("ssh user\@host")
Fleet::Session::expect($s, qr/password:/, 10)
Fleet::Session::send($s, "$pw\n")
Fleet::Session::close($s)

# Declarative playbook — first non-optional timeout fails the run
val $r = Fleet::Playbook::run("ssh user\@host", [
    @{ Fleet::Recipes::ssh_login(+{ password => $pw }) },
    +{ send   => "uptime\n" },
    +{ expect => qr/load average/, timeout => 15, name => "uptime output" },
    +{ send   => "exit\n" },
])
p $r->{ok} ? "done" : "failed: $r->{error}"
p "$_->{event} $_->{data}" for @{ $r->{transcript} }

# Branch tables — first match wins, optional action coderef
+{ branches => [
       +{ re => qr/\(yes\/no\)\?/, do => sub { "yes\n" } },
       +{ re => qr/password:/ },
   ],
   send_matched => 1 }

# The headline: one playbook, fifty hosts, parallel PTY sessions
val $results = Fleet::Fanout::ssh([@hosts],
    Fleet::Recipes::ssh_login(+{ password => $pw }))
val $part = Fleet::Fanout::partition($results)
p "ok: " . scalar(@{ $part->{ok} }) . "  failed: " . scalar(@{ $part->{failed} })
```

## [0x03] Sublibraries

| # | Module | File | Fns | Highlights |
|---|--------|------|----:|------------|
| 1 | `Fleet::Session` | `lib/Session.stk` | 17 | `open` · `send` · `send_line` · `expect` · `branch` · `read` · `buffer` · `alive` · `eof` · `interact` · `close` · `transcript` · `events` · `matches` · `last_match` · `timed_out` · `transcript_text` |
| 2 | `Fleet::Playbook` | `lib/Playbook.stk` | 3 | `validate` · `run` (step lists with `expect`/`send`/`branches`/`optional`/`send_matched`/`retries`/`retry_send`/`delay`) · `dry_run` (pure step preview) |
| 3 | `Fleet::Recipes` | `lib/Recipes.stk` | 36 | `ssh_login` · `sudo` · `su` · `psql` · `mysql` · `redis_cli` · `mongo` · `yes_to_all` · `cisco_enable` · `telnet_login` · `sftp` · `scp` · `ftp_login` · `passwd_change` · `docker_login` · `npm_login` · `gpg_decrypt` · `openssl_pem` · `rsync` · `ssh_keygen` · `git_clone` · `kinit` · `smbclient` · `vault_login` · `aws_configure` · `htpasswd` · `keytool` · `ansible_vault` · `sqlplus` · `op_signin` · `ssh_add` · `unzip_encrypted` · `cqlsh` · `gcloud_auth` · `heroku_login` · `names` |
| 4 | `Fleet::Fanout` | `lib/Fanout.stk` | 6 | `run` (N targets, `pmap`) · `ssh` (host-list convenience) · `partition` · `summarize` (counts + error lines) · `group_by_error` (bucket failures by error) · `retry_failed` (re-run only the failures) |

Recipes are pure data — they return playbook arrayrefs and never spawn anything, so they compose with `@{...}` splices and unit-test without a network.

## [0x04] What's NOT in Here

By design — these are stryke builtins, so we don't re-wrap them:

| Category | Builtins (call directly) |
|----------|--------------------------|
| PTY primitives | `pty_spawn` · `pty_send` · `pty_read` · `pty_expect` · `pty_expect_table` · `pty_buffer` · `pty_alive` · `pty_eof` · `pty_close` · `pty_interact` |
| Parallelism | `pmap` · `pgrep` · `ppool` and the rest of the parallel suite |
| Remote dispatch | `cluster([...])` + `pmap_on` — persistent SSH worker pools for *non-interactive* remote compute; Fleet is for sessions that prompt back |
| Method-form sugar | `PtyHandle` class (`require "perl_pty_class.stk"` from strykelang examples) |

If a function in this library can be replaced with one builtin call, it's a bug.

## [0x05] CLI

```sh
s bin/fleet.stk expect "sh -c 'sleep 1; echo ready'" ready 10   # spawn, wait, print match
s bin/fleet.stk exchange cat hello hello 5                      # spawn, send, wait
s bin/fleet.stk recipes                                         # list the recipe corpus
s bin/fleet.stk version
s bin/fleet.stk help
```

`bin/fleet.stk` covers one-shot expect/send loops from the shell. Anything with branches, retries, or more than two steps belongs in a playbook script — see `examples/`.

## [0x06] Tests

```sh
s test t/                       # assertions across every public function
```

`t/test_fleet.stk` is headless-CI safe by contract: every PTY assertion drives a local `cat`/`sh` — no network, no SSH, no target hosts (enforced by `tests/repo-contract.sh`). Recipes are asserted structurally as data.

## [0x07] Layout

```
stryke-fleet/
├── stryke.toml                # pure-stryke package manifest (no [ffi])
├── Makefile                   # test / install / clean
├── LICENSE                    # MIT
├── lib/
│   ├── Fleet.stk              # `use Fleet` — pulls all four sublibs
│   ├── Session.stk            # `use Fleet::Session`  — transcripted PTY sessions
│   ├── Playbook.stk           # `use Fleet::Playbook` — declarative step runner
│   ├── Recipes.stk            # `use Fleet::Recipes`  — login/prompt corpus
│   └── Fanout.stk             # `use Fleet::Fanout`   — parallel multi-target runs
├── bin/
│   └── fleet.stk              # CLI front-end (one-shot expect/exchange)
├── t/
│   └── test_fleet.stk         # all-surface assertions (local processes only)
├── examples/
│   ├── local_demo.stk          # runnable anywhere — drives a local sh
│   ├── installer_autopilot.stk # yes_to_all against a fake installer
│   ├── parallel_ssh.stk        # the headline: recipe × N hosts, partitioned
│   └── orchestrate.stk         # all four layers: recipe + playbook + transcript + fanout
├── tests/                     # shell gate scripts (CI lints)
└── docs/                      # GitHub Pages site
```

## [0xFF] License

MIT — see [LICENSE](LICENSE).
