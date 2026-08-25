# nettime

Like `time(1)`, but for network traffic: runs a command and reports how many
bytes it (and its children) sent and received, plus the peak rate.

```
$ sudo ./nettime -i 1 -o samples.csv node app.js

     real  1h04m11.83s
       rx  1.83 GiB
       tx  214.61 MiB
    total  2.04 GiB
     mean  4.55 Mbps
  peak@1s  94.31 Mbps  (at +0h12m44.02s)
```

Totals come from cumulative kernel counters read at start and end — not from
integrating rate samples — so they don't drift on long runs. Expect wire-level
(IP) bytes: headers, ACKs and retransmits included, so a few percent above
what the application itself counts. Sampling exists only to derive the peak
and to write the optional CSV, from which peak can be recomputed later over
any window.

## Usage

```
sudo ./nettime [-i seconds] [-o samples.csv] [-u user] COMMAND [ARGS...]
```

- `-i` sample interval for peak/CSV (default 1s)
- `-o` write `elapsed_s,rx_bytes,tx_bytes` samples to a CSV
- `-u` run COMMAND as this user (defaults to `$SUDO_USER` if invoked via
  sudo; useful with `su -c`, where `$SUDO_USER` is unset)

Root is required — the accounting below needs it. The command itself runs in
the foreground process group with your stdin, signals and exit status passed
through; `NETTIME_AS_ROOT=1` skips the de-escalation.

## How it works

Two mechanisms, picked automatically (`NETTIME_MECH=systemd|iptables`
overrides):

- **systemd hosts**: a transient scope with `IPAccounting=yes`; systemd's BPF
  program counts every byte of the scope's cgroup. Final totals are recovered
  from the journal after exit.
- **anything else** (OpenRC etc.): nettime makes its own cgroup2 leaf and
  counts egress with an `iptables -m cgroup --path` rule; ingress is counted
  by stamping a per-run conntrack mark (`0x4E54xxxx`) on the cgroup's
  connections and matching it on INPUT. The detour exists because plain
  cgroup matching on INPUT sees nothing on loopback. All rules and the cgroup
  are removed on exit, including on Ctrl-C.

Requirements for the second path: cgroup2 mounted at the kernel root,
iptables with the `xt_cgroup` match, and conntrack + `xt_connmark` for
ingress (modprobed automatically if needed). bash ≥ 5.1 is needed only for
accurate wall time on commands shorter than one interval.

## Caveats

- The conntrack mark overwrites any existing mark on the measured app's
  connections. If your host uses connmark-based routing or QoS, set
  `NETTIME_NO_CONNMARK=1` — loopback rx then reads 0.
- Not counted: the first inbound packet of connections the command accepts
  as a server (~60 B each), and inbound bytes on connections it never sends
  on. Unconnected-UDP ingress may be missed on the fallback path.
- **Docker is out of scope**: `nettime docker run ...` would measure the CLI
  talking to the daemon, not the container. For containers, poll
  `/proc/net/dev` inside the container's network namespace instead —
  `netlog.sh` in this repo does that.

## Status

The iptables path is tested end-to-end on one Linux 6.6 machine (accuracy
+0.2% over payload on loopback, +4% on a 1500-MTU TLS path; signals, exit
codes, stdin, child coverage and cleanup verified — details in `CONTEXT.md`).
The systemd path follows the documented interfaces and its parsing is
unit-tested, but it has not yet run against a live systemd. `netlog.sh` is
untested. Treat numbers from untested paths with suspicion, and expect no
better than a few percent from the tested one.
