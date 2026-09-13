# smokeping-mtr-alert-posix

When [SmokePing](https://oss.oetiker.ch/smokeping/) detects an [alert](https://oss.oetiker.ch/smokeping/doc/smokeping_config.en.html#___top) condition it can run a program instead of sending its own e-mail.

This script, launched by SmokePing in "pipe" mode, runs an [MTR](https://www.bitwizard.nl/mtr/) traceroute in report mode and mails the output to the people SmokePing would have mailed anyway, together with a link to — and an inline image of — the graph that triggered the alert. When the target is measured by a remote probe, the traceroute is run **on that probe's host**, over the same transport the probe itself uses, so the path in the mail is the path where the loss actually is.

There is nothing to configure and no wrapper script to write: every setting is read from the SmokePing configuration itself.

## Credits

This is a rewrite in pure shell of [`catalyst/smokeping-mtr-alert`](https://github.com/catalyst/smokeping-mtr-alert) by **Michael Fincham `<michael.fincham@catalyst.net.nz>`**, Copyright (c) 2016 **Catalyst.net Ltd**. The original is Python 2; the idea, the message layout and the argument handling are his and are kept recognisable here. Licensed, like the original, under the GNU General Public License v3 or later.

### What is different from the original

| | original | this rewrite |
|---|---|---|
| Interpreter | Python 2 (`pipes`, `str`/`bytes` breakage on Python 3) | POSIX `sh` — runs under dash, ash, busybox, bash |
| Configuration | wrapper script with `--email` / `--name` | none; read from the SmokePing configuration |
| Recipients | one hardcoded address | `Alerts/to`, the alert's own `to`, the target's inherited `alertee`, else `General/contact` |
| Remote probes | traceroute always from the SmokePing host | traceroute from the probe's `rhost`, using the probe's own `binary`, `ruser` and `rport` |
| Graph URL in the mail | no | yes, the target's page in the web UI |
| Graph image in the mail | no | inline PNG in a `multipart/related` message |
| `edgetrigger = yes` | unsupported, aborts on the 6th argument | supported, raise/clear reflected in the subject |
| MTR failure | alert silently dropped | error and exit status included in the mail |
| MTR runtime | unbounded | bounded by `timeout(1)` |
| Headers | `From`, `To`, `Subject` | plus `Date`, `Message-ID`, UTF-8, RFC 2047, `Auto-Submitted`, `Precedence`, `X-SmokePing-*` |
| Delivery | `/usr/sbin/sendmail` | `General/mailhost` over SMTP when set, else `General/sendmail`, else autodetected |

## Dependencies

* `mtr` — on the SmokePing host, and on the host of any remote probe, with raw socket privileges (see [Troubleshooting](#troubleshooting)).
* `rrdtool` — the command line binary. SmokePing itself uses the `RRDs` Perl bindings, so on some installations the binary is not pulled in and has to be added explicitly. Without it the mail is sent without the graph.
* a sendmail-compatible MTA, unless `General/mailhost` is set, in which case `curl` is used to talk SMTP.
* `sh`, `sed`, `awk`, `date`, `base64`, `hostname`/`uname`, and optionally `timeout` from coreutils.

No Perl, Python or Ruby is involved.

## Installation

```sh
install -m 0755 smokeping-mtr-alert /usr/local/bin/
dash -n /usr/local/bin/smokeping-mtr-alert   # silence means it parses
```

The file is plain ASCII with LF line endings and no BOM. If it has been through a Windows machine, a browser download or a copy-paste on the way to the server, strip the carriage returns before anything else — `/bin/sh^M: bad interpreter` is the symptom, and `^M` at the end of every other line breaks things far less visibly:

```sh
sed -i 's/\r$//' /usr/local/bin/smokeping-mtr-alert
```

The script must be readable and executable by the user SmokePing runs as (`smokeping` on Debian/Ubuntu, `smokeping` or `daemon` on EL), and that user must be able to read the SmokePing configuration.

Then configure SmokePing in "pipe" mode. The `from` address is used as the envelope sender and in the `From` header:

```
*** Alerts ***
to = |/usr/local/bin/smokeping-mtr-alert
from = smokeping@example.com

+someloss
type = loss
# in percent
pattern = >0%,*12*,>0%,*12*,>0%
comment = loss 3 times in a row
```

Attach the alert to targets as usual:

```
++ Router1

menu = Router1
title = Core router 1
host = 192.0.2.1
alerts = someloss
```

`edgetrigger = yes` is supported: SmokePing then passes a sixth argument (`1` on raise, `0` on clear), the subject becomes `... alert raised:` or `... alert cleared:`, and a `State:` line is added to the body.

More information about the alerting syntax is available [in the SmokePing documentation](https://oss.oetiker.ch/smokeping/doc/smokeping_config.en.html#___top).

## Where each setting comes from

Nothing is configurable in the script and there is no configuration file of its own. Everything is read from the SmokePing configuration, `@include` directives followed, at `/etc/smokeping/config` or, failing that, `/usr/local/smokeping/etc/config`, `/usr/local/etc/smokeping/config`, `/opt/smokeping/etc/config` or `/etc/smokeping.conf`. Set `SMOKEPING_CONFIG` in the environment for an installation that lives somewhere else.

| What | Read from |
|---|---|
| Recipients | `Alerts/to` + the alerting `+alert`'s own `to` + the target's inherited `alertee`, de-duplicated, with the pipe and any `snpp:`/`xmpp:` entries dropped; `General/contact` if that yields nothing |
| Sender | `Alerts/from`, displayed as `General/owner` (or `display_name`) |
| Installation name | `General/display_name`, else `General/owner`, else the local hostname |
| Graph link | `General/cgiurl` |
| Graph data | `General/datadir`, plus `Database/pings` when the RRD cannot be read |
| Graph size | `Presentation/detail/width` and `height` |
| Graph time range | the first row of the `Presentation/detail` range table, so the embedded image covers the same window the CGI opens on |
| Which probe measured the target | the target's inherited `probe` in the Targets tree |
| Where that probe measures from | that probe's `rhost`, `ruser`, `rport` and `binary` in the Probes section |
| Delivery | `General/mailhost` (comma separated, with `mailuser`/`mailpass`) over SMTP, else `General/sendmail`, else `/usr/sbin/sendmail`, `/usr/lib/sendmail` or `sendmail` on the `PATH` |

Because the recipients are worked out the same way SmokePing works them out, adding an `alertee` to a target or a `to` to an alert changes who gets the MTR report too, with no second place to keep in sync.

The only thing set in the script is the MTR command line, at the top of the file:

```sh
MTR_OPTS="-w --report -z --show-ips -c 20"
MTR_TIMEOUT=180
```

## Remote probes

A probe that measures from somewhere else — `RemoteFPing` and anything else configured with an `rhost` — carries everything needed to reach that host in the Probes section:

```
*** Probes ***
+ RemoteFPing
binary = /usr/bin/ssh
rbinary = /usr/sbin/fping

++ RemoteFpingSite2
rhost = probe2.example.com
ruser = smokeping
rport = 2222
```

The script finds the target's effective `probe` by walking the Targets tree the way SmokePing inherits it, looks that probe up in the Probes section — a probe named on a target is either a module section or an instance under one, and an instance inherits the module section's settings — and, if it has an `rhost`, runs the traceroute there with exactly the call the probe itself uses:

```
binary [-l ruser] [-p rport] rhost mtr <MTR_OPTS> -- <target host>
```

So if fping already runs on that host for this target, mtr will too: same binary, same user, same port, same keys, nothing new to configure. `-n -o BatchMode=yes -o ConnectTimeout=10` is added when the binary looks like ssh, so a host that asks for a password fails immediately instead of hanging the alert.

The body names the vantage point either way:

```
Measured by: probe RemoteFpingSite2 on probe2.example.com
Measured by: probe FPing on smokeping.example.com
```

If the remote call fails the alert is still sent: the traceroute is re-run locally and the message says so, quoting the error, with `Measured by: smokeping.example.com (probe RemoteFpingSite2 on probe2.example.com could not be reached)` so nobody reads a local path as if it were the probe's.

**`mtr` must be installed on the probe's host**, with raw socket privileges for `ruser`. Check with the same call the probe uses: `sudo -u smokeping ssh -l smokeping -p 2222 probe2.example.com mtr --version`.

### Distributed SmokePing

SmokePing also has a distributed mode, configured under `*** Slaves ***`, where other SmokePing instances report their measurements back to a master. That is a different mechanism from a remote probe and most installations never use it. When it is in use the alert's target carries a `[from <name>]` suffix and the reporting instance's samples live in their own `<target>~<name>.rrd`; the script handles both, but a normal installation never produces either.

## The graph

The message is `multipart/related` wrapping a `multipart/alternative`: a plain text part, an HTML part, and the graph as an inline PNG referenced by `cid:`. Clients that refuse HTML still get the full text version, and the image is shown in the body rather than dangling as an attachment. If the graph cannot be produced — no rrdtool, no RRD yet for that target — the message falls back to plain text, so a missing graph never costs you the alert.

One target is one RRD: the target path with its dots turned into directory separators under `datadir`, so `Backbone.Router1` is `/var/lib/smokeping/Backbone/Router1.rrd`. The file is checked for rather than assumed, and the one that is found is recorded in an `X-SmokePing-RRD` header. If it is not there the message still goes out, without the graph, and the warning on stderr lists what that directory really contains — so a wrong `datadir` or an unexpected layout is visible rather than guessed at.

It is rendered locally with `rrdtool graph`, the way SmokePing's CGI does it: the number of pings comes from the RRD's data sources, the grey "smoke" bands between the fastest and slowest ping of each sample are a port of `Smokeping::smokecol()`, and the median line is split into SmokePing's default loss colour buckets (`#26ff00` for no loss through `#a00000` for total loss), with the usual median and packet loss legend underneath. Size and time range come from `Presentation/detail`, so the image matches what the web UI would have shown.

Drawing from the RRD rather than fetching from the CGI means no HTTP, no authentication and no dependence on the CGI being reachable — and it sidesteps SmokePing 2.9.0, which switched graphs from PNG to SVG; no mail client renders SVG reliably.

## Example e-mail

```
From: "Network Operations" <smokeping@example.com>
To: noc@example.com
Subject: Network Operations SmokePing alert: Backbone.Router1
Date: Mon, 14 Sep 2026 00:03:27 +0000
MIME-Version: 1.0
Auto-Submitted: auto-generated
Precedence: bulk
X-SmokePing-Alert: someloss
X-SmokePing-Target: Backbone.Router1
X-SmokePing-Probe: RemoteFpingSite2 on probe2.example.com
X-SmokePing-RRD: /var/lib/smokeping/Backbone/Router1.rrd

Packet loss report from Network Operations for Backbone.Router1 at Mon Sep 14 00:03:27 2026.
Measured by: probe RemoteFpingSite2 on probe2.example.com

http://smokeping.example.com/smokeping.cgi?target=Backbone.Router1

ssh -l smokeping -p 2222 probe2.example.com mtr -w --report -z --show-ips -c 20 192.0.2.1

Start: Mon Sep 14 00:03:06 2026
HOST: probe2                                    Loss%   Snt   Last   Avg  Best  Wrst StDev
  1. AS64500  gw1.example.com (198.51.100.1)     0.0%    20    0.3   0.4   0.2   0.9   0.1
  2. AS64500  core1.example.net (203.0.113.9)    0.0%    20    2.0   2.1   1.5   3.8   0.3
  3. AS???    ???                               100.0    20    0.0   0.0   0.0   0.0   0.0

Alert triggered: someloss
Target: Backbone.Router1
Target hostname: 192.0.2.1
Loss pattern: 0%, 0%, 100%
RTT: 12ms, 13ms, U
```

Carried as the text part of:

```
multipart/related; type="multipart/alternative"
├── multipart/alternative
│   ├── text/plain            the message above
│   └── text/html             the same, with <img src="cid:...">
└── image/png                 Content-Disposition: inline, Content-ID: <...>
```

Header words that come out of the configuration are RFC 2047 encoded when they are not ASCII, so a non-ASCII `owner` does not corrupt the `From` line.

## Usage

The positional arguments are supplied by SmokePing itself.

```
usage: smokeping-mtr-alert [-n] <alert> <target> <loss-pattern> <rtt-pattern> <hostname> [state]

Called by SmokePing as a pipe alert:

  *** Alerts ***
  to = |/usr/local/bin/smokeping-mtr-alert
  from = smokeping@example.org

All settings come from the SmokePing configuration. Set SMOKEPING_CONFIG in the
environment if it is not in one of the usual places.

  -n, --dry-run   print the message on stdout instead of sending it
  -h, --help      this text
```

### Testing

`--dry-run` prints the complete message on stdout and never touches the MTA, so the whole path — config parsing, recipients, the call out to the probe's host, the graph — can be checked before an alert ever fires. Use a real target path, and run it as the SmokePing user so file permissions and ssh keys are exercised as they will be in production:

```sh
sudo -u smokeping /usr/local/bin/smokeping-mtr-alert -n \
  someloss Backbone.Router1 "loss: 0%, 0%, 100%" "rtt: 12ms, 13ms, U" 192.0.2.1
```

Drop `-n` to send it for real. The `X-SmokePing-Probe` and `X-SmokePing-RRD` headers show what the script worked out from the configuration, and warnings about anything it could not work out go to stderr, where SmokePing would discard them.

## Troubleshooting

**`/bin/sh^M: bad interpreter`** — the file picked up CRLF line endings in transit. `sed -i 's/\r$//'` on it; see [Installation](#installation).

**`no readable SmokePing config found`** — the config is not in any of the usual paths. Set `SMOKEPING_CONFIG` in the environment of the SmokePing process, or symlink it.

**`no recipient: no mail address in Alerts/to, alertee or General/contact`** — `Alerts/to` contains only the pipe to this script and nothing else defines an address. Either add one there (`to = |/usr/local/bin/smokeping-mtr-alert,noc@example.com`), give the target an `alertee`, or set `General/contact`.

**`error running MTR ...: Failure to start mtr-packet`** — SmokePing runs alert programs as its own unprivileged user and mtr needs raw sockets. On Debian and Ubuntu, `setcap cap_net_raw+ep /usr/bin/mtr-packet`; on EL the binaries live in `/usr/sbin`, so apply the same capability to `/usr/sbin/mtr-packet` (or to `mtr` itself on builds without a separate packet helper). This applies on remote probe hosts too. Verify with `sudo -u smokeping mtr -n --report 192.0.2.1`.

**`could not run MTR on <rhost>`** — run the probe's own call by hand as the SmokePing user: `ssh -l <ruser> -p <rport> <rhost> mtr --version`. If fping works there and mtr does not, mtr is either not installed on that host or lacks raw socket privileges for `ruser`.

**The trace runs locally when it should be remote** — check `X-SmokePing-Probe` in the message. If it names no probe, the target has no `probe` on it or above it in the Targets tree; if it names one with no `on <host>`, that probe has no `rhost` and genuinely does measure locally.

**`no graph embedded, sending text only`** — the line above it says which step failed. `no RRD for <target> in <dir>` lists what that directory actually holds: if the file you expect is there under another name, the target path in the alert and the on-disk layout disagree; if the directory is empty or missing, `General/datadir` is not where the RRDs live. Missing `rrdtool` is the other common cause.

**`@define is not expanded`** — the script reads the configuration directly and does not implement Config::Grammar's `@define` macros. Values that depend on one will be read literally. Nothing else in the parse is affected.

**No mail at all** — SmokePing double-forks alert programs and their output follows SmokePing's own stderr, which is `/dev/null` once daemonised, so failures are invisible. Run the command by hand with `--dry-run` first, then check the MTA queue and logs. `to` must begin with `|` in the SmokePing config or the script is never called.

**SELinux (EL)** — a confined SmokePing may be denied outbound mail, raw sockets or ssh. Check `ausearch -m avc -ts recent` before assuming the script is at fault.

## Limitations

* One MTR and one `rrdtool graph` per alert. On a wide outage that is one of each per target; use `edgetrigger = yes` and alert `priority` to keep the volume sane.
* MTR runs *after* the loss has been detected, so a transient event may have cleared by the time the traceroute runs.
* Probes that measure from a network device rather than a host — `OpenSSHJunOSPing`, `TelnetIOSPing` and friends — have no `rhost`, so their alerts trace locally. There is no mtr on a router to call.
* Config::Grammar `@define` macros are not expanded.

## License

GNU General Public License v3 or later, inherited from the original work by Michael Fincham / Catalyst.net Ltd.
