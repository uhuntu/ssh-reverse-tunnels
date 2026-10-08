# SSH reverse tunnels that don't die

A field guide to running `ssh -R` forwards as a service: how to keep them alive,
how to stop them fighting your interactive sessions, and how to tell who
actually owns the port when something breaks.

Everything here is derived from a setup that carried RDP, SSH and HTTP-proxy
forwards across two jump hosts for months, including the crash-loop bug that
forced the design.

---

## 1. Who owns the listening socket

This is the single most useful thing to internalise, because it determines what
you see when you debug.

| Flag | Listener is created by | You see it on |
|---|---|---|
| `-L 7897:host:port` | the **local `ssh` client** | the client box, owned by `ssh` |
| `-R 7897:host:port` | the **remote `sshd`** | the server box, owned by `sshd` |

So on the *remote* machine:

```bash
$ sudo ss -ltnp | grep 7897
LISTEN 0 128 127.0.0.1:7897 0.0.0.0:*  users:(("sshd",pid=1490095,fd=7))
```

**That `sshd` is not your proxy.** It is the tunnel that a client asked for.
Pointing an application at `127.0.0.1:7897` means "go through whatever proxy is
listening on `host:port` on the *other* end of someone's SSH session".

Confirming it cheaply: compare egress IPs.

```bash
curl -s --noproxy '*' -x http://127.0.0.1:7890 https://api.ipify.org
curl -s --noproxy '*' -x http://127.0.0.1:7897 https://api.ipify.org
```

If the two differ, they are two different proxies, no matter what any
config file claims.

### The practical consequence

`-R` forwards are **only alive while the owning SSH connection is alive**. When
that connection drops, the port closes and everything pointed at it fails with
`ECONNREFUSED`. An interactive session's forward is therefore the wrong thing to
depend on. It has to be owned by something with a restart policy.

---

## 2. The crash-loop: `ControlPersist` + `ssh -NT` + systemd

The bug that forces the whole design, in one paragraph.

You want a persistent tunnel, so you write a systemd user unit:

```ini
[Service]
ExecStart=/usr/bin/ssh -NT jump-tunnel
Restart=always
```

and the `jump-tunnel` host block in `~/.ssh/config` also carries the multiplexing
options your interactive sessions use:

```
Host jump-tunnel
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h-%p
  ControlPersist 30m
  RemoteForward 127.0.0.1:7897 localhost:19674
```

It appears to work. It does not.

`ControlPersist 30m` makes `ssh` fork a **background master** and the foreground
process exit `0` immediately. systemd sees `ExecStart` exit cleanly, considers
the unit done, and tears down the unit's cgroup — killing the master it just
forked. Repeat every few seconds.

```bash
$ systemctl --user show reverse-tunnel -p NRestarts
NRestarts=23402
```

If a *manually* launched `ssh -M -N -f` happens to hold the binds at the same
time, the forwards still work, and you will wrongly conclude the service is
fine. Remove that crutch and everything breaks.

### The fix

**Never put `ControlMaster`/`ControlPath`/`ControlPersist` in a host block that
a systemd unit uses.** Split the block in two.

---

## 3. The design: split the alias

Two aliases per jump host, with completely disjoint jobs.

### A. The tunnel alias — used *only* by the systemd unit

```
Host jump-tunnel
  HostName 203.0.113.10
  User tunnel
  BatchMode yes
  RemoteForward 127.0.0.1:7897  localhost:19674
  RemoteForward 127.0.0.1:3389  localhost:3389
  RemoteForward 127.0.0.1:3390  localhost:3390
  RemoteForward 127.0.0.1:2222  localhost:22
  ExitOnForwardFailure yes
  ServerAliveInterval 30
  ServerAliveCountMax 10
```

No multiplexing options at all. `BatchMode` so it can never block on a password
prompt and hang the unit.

**`ExitOnForwardFailure yes` is the important one.** Without it, `ssh` will
happily report success even though it failed to bind, and you get a unit that is
`active` and a set of ports that are `closed`. With it, a bind conflict makes
`ssh` exit non-zero → systemd restarts → the failure is loud instead of silent.

The cost: if *anything else* already holds one of those ports, the tunnel alias
refuses to start. Which brings us to the second alias.

### B. The interactive alias — multiplexing, no forwards

```
Host 203.0.113.10
  HostName 203.0.113.10
  User tunnel
  BatchMode yes
  ServerAliveInterval 30
  ServerAliveCountMax 10
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h-%p
  ControlPersist 30m
```

Zero `RemoteForward` lines. Interactive login stays instant and shared, and can
never collide with the service's binds. Before the split, logging in could kill
the tunnel and vice versa.

### C. Extra purpose-specific aliases, with their own `ControlPath`

If you sometimes want a forward that the service does *not* own, give it its own
alias **and its own control path**, otherwise the two masters fight over binds:

```
Host jump-proxy
  HostName 203.0.113.10
  User tunnel
  BatchMode yes
  RemoteForward 127.0.0.1:10808 localhost:19674
  ExitOnForwardFailure yes
  ServerAliveInterval 30
  ServerAliveCountMax 10
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h-proxy-%p
  ControlPersist 30m
```

Note the `-proxy-` in the `ControlPath`. Two aliases to the *same* host with the
*default* control path share one master, which means one alias's forwards can be
killed by the other's expectations. Same reasoning as why the main alias
deliberately omits one of the ports in this example.

---

## 4. The systemd unit

```ini
# ~/.config/systemd/user/reverse-tunnel.service
[Unit]
Description=SSH reverse tunnel (managed forwards)
After=network-online.target

[Service]
Type=simple
# -F is redundant if the alias lives in ~/.ssh/config, but being explicit
# documents exactly which block the unit depends on.
ExecStart=/usr/bin/ssh -NT jump-tunnel
Restart=always
RestartSec=5
StartLimitIntervalSec=0

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now reverse-tunnel.service
journalctl --user -u reverse-tunnel.service -f
```

`StartLimitIntervalSec=0` disables the start rate limit; combined with
`Restart=always` you want the unit to keep retrying forever rather than give up
and enter `failed` after five attempts. That "give up" behaviour is what turns a
transient network blip into a week-old dead tunnel.

---

## 5. Debugging: who owns this port, really?

Ordered from cheapest to most invasive. All of these work unprivileged except
where noted.

### 5.1 Who is listening

```bash
sudo ss -ltnp | grep 7897          # needs sudo for the pid
ss -ltne  | grep 7897              # no sudo: gives you uid + inode
```

If `ss -ltnp` shows a port with no `pid=`, the owning process is root's. Do not
conclude "orphan" — see 5.4.

### 5.2 Map inode → pid without sudo

`ss -ltne` prints `ino:<n>`. Then:

```bash
for p in $(pgrep -u "$USER" .); do
  ls -l /proc/$p/fd 2>/dev/null | grep -q "socket:\[$ino\]" && ps -o pid,args -p $p
done
```

Only covers your own uid. Root-owned processes are invisible here.

### 5.3 Read the process tree

```bash
ps -eo pid,ppid,user,etimes,args | grep sshd
```

Worth knowing the shapes:

| Process title | Meaning |
|---|---|
| `sshd: user@pts/3` | interactive session on that pty |
| `sshd: user@notty` | session requested, no tty (`ssh host cmd`, `ssh -T`) |
| `sshd: user` | no session channel at all → **client used `-N`** |
| `sshd: user [priv]` | the privileged parent of one connection |

Multiple ptys in one entry (`sshd: hunt@pts/2,pts/5,pts/6`) means one connection
multiplexing several sessions.

Cgroup tells you *who launched it*:

```bash
cat /proc/<pid>/cgroup
```

`0::/user.slice/user-1001.slice/session-14956.scope` — a logind session scope.
A systemd user service's own processes normally sit under that service's cgroup.
Caveat: `pam_systemd` moves per-connection sshd processes into a session scope
regardless of who launched them, so this is suggestive, not conclusive.

### 5.4 Why `ss -ltnp` shows nothing and `/proc/<pid>/fd` is empty

sshd clears `PR_SET_DUMPABLE` on its unprivileged children, so as a normal user:

```
$ ls -l /proc/1490095/fd
ls: cannot open directory '/proc/1490095/fd': Permission denied
```

Even though the process runs as you. Your inode scan from 5.2 silently skips it.
Use `sudo` and it appears immediately. This exact misdiagnosis — "an unknown
process holds the port, probably an orphan" — is easy to make and wrong.

### 5.5 Is the tunnel actually my proxy?

Compare egress IPs, and probe whether the far end speaks your proxy protocol:

```bash
curl -s --noproxy '*' -x http://127.0.0.1:7897 https://api.ipify.org
curl -s --noproxy '*' -x socks5h://127.0.0.1:7897 https://api.ipify.org
```

A Clash-style `mixed-port` answers both; a plain HTTP forward answers only the
first.

### 5.6 Airport/CDN gotchas when the far end is a proxy subscription

If the tunnel's target is an HTTP proxy that fronts a CDN, the *subscription*
fetch may depend on User-Agent. BunnyCDN-fronted subscriptions commonly return
`403` plus a fake Clash config to unknown UAs:

```yaml
proxies:
  - name: "访问受限 REF: ..."
    type: trojan
    server: 127.0.0.1
    port: 1
    password: blocked
```

A `trojan` entry pointing at `127.0.0.1:1` is always the tell. When adding such
a subscription to mihomo, pin the UA on the provider:

```yaml
proxy-providers:
  myisp:
    type: http
    url: 'https://example.com/subscribe'
    interval: 3600
    header:
      User-Agent:
      - clash-verge/2.0.0
```

---

## 6. Checklist

- [ ] `-R` forwards are owned by a systemd unit, never by a login session
- [ ] The unit's host alias has **no** `ControlMaster`/`ControlPath`/`ControlPersist`
- [ ] The unit's host alias has `ExitOnForwardFailure yes`
- [ ] Interactive alias multiplexes and forwards nothing
- [ ] Any extra-purpose alias has its own `ControlPath`
- [ ] `Restart=always` + `StartLimitIntervalSec=0`
- [ ] `BatchMode yes` everywhere a unit is involved
- [ ] Nothing in `~/.ssh/config`, systemd units, or app config contains a
      real secret or internal address before it goes anywhere public

---

## Files

```
examples/ssh_config           sanitised, annotated
examples/reverse-tunnel.service
```

Real IPs, hostnames, subscription tokens and node passwords have been replaced
with RFC 5737 documentation addresses (`203.0.113.0/24`, `198.51.100.0/24`) and
placeholders. Do not paste your own config anywhere without the same treatment.

## License

MIT — see [LICENSE](LICENSE).
