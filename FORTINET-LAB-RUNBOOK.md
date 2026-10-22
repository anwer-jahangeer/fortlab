# Fortinet Dual-ISP Lab Runbook

Last updated: October 21, 2026

## 1. Purpose

This lab validates FortiGate dual-ISP VPN behavior during controlled ISP2 packet loss and latency. It generates continuous branch traffic, applies scheduled WAN impairment, and records baseline and impaired internet performance from every site.

All schedules use the `America/Chicago` timezone. Times automatically follow Central Standard Time and Central Daylight Time.

## 2. Topology and Access

### FortiGate appliances

| Site | Management / WAN1 IP | Role |
|---|---:|---|
| Hub1 | `10.10.111.20` | VPN hub and traffic destination |
| Hub2 | `10.10.112.20` | VPN hub and traffic destination |
| Branch1 | `10.10.113.20` | Dual-ISP branch |

### Linux test hosts

| Site | Management IP | Service/source IP | SSH user | Hostname used by automation |
|---|---:|---:|---|---|
| Hub1 | `10.10.252.31` | `10.10.20.6` | `ubuntu` | `ip-10-10-20-6` |
| Hub2 | `10.10.252.32` | `10.10.30.6` | `ubuntu` | `hub2` |
| Branch1 | `10.10.252.33` | `10.10.10.6` | `ubuntu` | `ip-10-10-10-6` |
| Branch2 | `10.10.252.34` | `10.10.40.6` | `ubuntu` | `branch2` |

### Internet gateway

| Item | Value |
|---|---|
| Management IP | `10.10.121.10` |
| SSH user | `admin` |
| OS | Debian Linux |
| Timezone | UTC; schedules explicitly use `America/Chicago` |
| Authentication | Public key only |

### Windows administration files

| Item | Path |
|---|---|
| PuTTY private key | `C:\Users\Administrator\Desktop\fortinet-putty.ppk` |
| Plink | `C:\Users\Administrator\Tools\PuTTY\plink.exe` |
| PSCP | `C:\Users\Administrator\Tools\PuTTY\pscp.exe` |
| Local scripts and this runbook | `C:\Users\Administrator\Tools\PuTTY` |

Example SSH connection from PowerShell:

```powershell
C:\Users\Administrator\Tools\PuTTY\plink.exe `
  -ssh -i C:\Users\Administrator\Desktop\fortinet-putty.ppk `
  ubuntu@10.10.252.32
```

Use `admin@10.10.121.10` for the internet gateway.

## 3. Completed FortiGate Work

### Hub2 license

The correct license file was `C:\tftp\FGVMSLTM26036731.lic`. It was uploaded to FortiOS using:

```text
/api/v2/monitor/system/vmlicense/upload
```

FortiGuard transport was changed so the license could activate:

```fortios
config system fortiguard
    set fortiguard-anycast disable
    set protocol udp
    set port 8888
end
```

Final state: `License Status: Valid`.

### Branch1 VPN2 repair

Branch1 VPN2 tunnels were down because SD-WAN member `port3` did not have a gateway. The repair was:

```fortios
config system sdwan
    config members
        edit 2
            set gateway 10.10.103.1
        next
    end
end
```

After the change, all four branch VPN tunnels and associated BGP neighbors were validated as up.

## 4. Internet Gateway Interface Mapping

| Site/link | IGW interface | Impairment method |
|---|---|---|
| Hub1 ISP1 | `ens6` | None |
| Hub1 ISP2 | `ens7` | Direct netem |
| Hub2 ISP1 | `ens8` | None |
| Hub2 ISP2 | `ens9` | Direct netem |
| Branch1 ISP1 | `ens10` | None |
| Branch1 ISP2 | `ens11` | Direct netem |
| Branch2 ISP1 | `ens12` | None |
| Branch2 ISP2 | `ens5` | NAT-aware filters and netem |

Branch2 ISP2 uses:

```text
OUTSIDE_MAC=02:38:f8:2b:2f:03
BRANCH2_ISP2_PUBLIC_IP=3.137.69.191
```

These values are stored on the IGW in `/etc/default/fortinet-branch2`.

## 5. Impairment Profile

Only ISP2 is impaired.

| Setting | Value |
|---|---:|
| Added delay | `500ms` |
| Packet loss | `10%` |
| Direction | Bidirectional |
| Technology | Linux `tc` / `netem` with IFB ingress redirection |

Direct interfaces use:

- `/usr/local/sbin/set-site-netem.sh`
- `/usr/local/sbin/clear-site-netem.sh`

Branch2 uses NAT-aware scripts:

- `/usr/local/sbin/set-branch2-isp2-netem.sh`
- `/usr/local/sbin/clear-branch2-isp2-netem.sh`

The Branch2 IFB is named `ifb-br2-isp2`. Linux interface names are limited to 15 characters; the earlier longer name was invalid.

## 6. Daily Central Time Schedule

### Full daily timeline

| Time CT | Event |
|---:|---|
| 8:30 AM | Hub1 baseline internet iPerf and Speedtest |
| 8:55 AM | Start continuous traffic on Branch1 and Branch2 |
| 9:00 AM | Start Hub1 ISP2 impairment |
| 9:30 AM | Hub1 impaired internet iPerf and Speedtest |
| 10:00 AM | Clear Hub1 impairment |
| 10:30 AM | Hub2 baseline internet iPerf and Speedtest |
| 11:00 AM | Start Hub2 ISP2 impairment |
| 11:30 AM | Hub2 impaired internet iPerf and Speedtest |
| 12:00 PM | Clear Hub2 impairment |
| 12:30 PM | Branch1 baseline internet iPerf and Speedtest |
| 1:00 PM | Start Branch1 ISP2 impairment |
| 1:30 PM | Branch1 impaired internet iPerf and Speedtest |
| 2:00 PM | Clear Branch1 impairment |
| 3:30 PM | Branch2 baseline internet iPerf and Speedtest |
| 4:00 PM | Start Branch2 ISP2 impairment |
| 4:30 PM | Branch2 impaired internet iPerf and Speedtest |
| 5:00 PM | Clear Branch2 impairment |
| 5:05 PM | IGW safety-clears all impairment; stop continuous branch traffic |
| 5:10 PM | Safety-stop continuous branch traffic again |

### Impairment windows

| Site | Daily ISP2 impairment window CT | IGW timers |
|---|---|---|
| Hub1 | 9:00-10:00 AM | `netem-hub1-start.timer`, `netem-hub1-stop.timer` |
| Hub2 | 11:00 AM-12:00 PM | `netem-hub2-start.timer`, `netem-hub2-stop.timer` |
| Branch1 | 1:00-2:00 PM | `netem-branch1-start.timer`, `netem-branch1-stop.timer` |
| Branch2 | 4:00-5:00 PM | `netem-branch2-start.timer`, `netem-branch2-stop.timer` |
| All sites | Safety clear at 5:05 PM | `netem-safety-clear.timer` |

### Benchmark times

| Site | Baseline CT | Impaired CT |
|---|---:|---:|
| Hub1 | 8:30 AM | 9:30 AM |
| Hub2 | 10:30 AM | 11:30 AM |
| Branch1 | 12:30 PM | 1:30 PM |
| Branch2 | 3:30 PM | 4:30 PM |

Each host uses:

- `daily-site-benchmark-baseline.timer`
- `daily-site-benchmark-impaired.timer`
- `site-benchmark@baseline.service`
- `site-benchmark@impaired.service`

## 7. Continuous Branch Traffic

Continuous traffic runs daily from 8:55 AM until 5:05 PM CT, with a second safety stop at 5:10 PM.

Branch1 and Branch2 each generate:

| Traffic | Destination | Frequency/rate |
|---|---|---|
| Ping | Hub1 `10.10.20.6` | One ping every 5 seconds |
| Ping | Hub2 `10.10.30.6` | One ping every 5 seconds |
| Ping | Internet `1.1.1.1` | One ping every 5 seconds |
| HTTPS | `https://example.com` | Every 10 seconds |
| Restricted SSH | Both hubs | Every 10 seconds |
| UDP iPerf3 | Both hubs | `2 Mbps`, 50-second test, 10-second pause |

Branch1 uses hub iPerf port `5201`. Branch2 uses hub iPerf port `5202`, allowing both branches to test concurrently.

The SSH sessions use `/home/ubuntu/.ssh/lab_traffic_ed25519`. Hub authorization is restricted with `restrict,command="/bin/true"`, preventing interactive shells, arbitrary commands, and forwarding.

Traffic services and timers:

- `lab-traffic.service`
- `daily-lab-traffic-start.timer`
- `daily-lab-traffic-stop.timer`
- `daily-lab-traffic-safety-stop.timer`

## 8. Twice-Daily Internet Benchmarks

Every site runs one baseline and one impaired benchmark each day.

The benchmark performs:

1. Route lookup for `1.1.1.1`.
2. A 15-second TCP upload iPerf test to a public internet server.
3. A 15-second reverse/download iPerf test.
4. An internet Speedtest using `speedtest-cli --secure --simple`.

Public iPerf endpoints are attempted in this order:

1. `iperf3.moji.fr:5202`
2. `iperf.he.net:5201`
3. `speedtest1.serverius.net:5002`

A public iPerf endpoint is considered successful only if both upload and download tests complete. Public servers commonly allow only one client per port, so a server can be temporarily busy or reset a connection. The script automatically tries the next endpoint.

The old private benchmark listener on TCP port `5210` was removed from both hubs. Continuous branch-to-hub listeners on ports `5201` and `5202` remain active.

## 9. Logs and Retention

### Continuous traffic logs

Stored on Branch1 and Branch2 under:

```text
/var/log/lab-traffic/<hostname>/
```

Important files include:

```text
ping-hub1.log
ping-hub2.log
ping-internet.log
https-internet.log
ssh-hub1.log
ssh-hub2.log
iperf-hub1.log
iperf-hub2.log
```

Logrotate configuration: `/etc/logrotate.d/lab-traffic`

Retention: 14 daily compressed rotations using `copytruncate`.

### Benchmark logs

Stored on every test host under:

```text
/var/log/site-benchmark/
```

Files are labeled by site and phase, for example:

```text
hub2-baseline-summary.log
hub2-baseline-internet-iperf.log
hub2-baseline-speedtest.log
hub2-impaired-summary.log
hub2-impaired-internet-iperf.log
hub2-impaired-speedtest.log
```

Logrotate configuration: `/etc/logrotate.d/site-benchmark`

Retention: 14 daily compressed rotations.

## 10. Daily Health Checks

### Check all IGW impairment timers

Connect to `admin@10.10.121.10` and run:

```bash
systemctl list-timers 'netem-*' --all
systemctl --failed
```

### Check benchmark timers on a test host

```bash
systemctl list-timers 'daily-site-benchmark-*' --all
systemctl is-enabled daily-site-benchmark-baseline.timer \
  daily-site-benchmark-impaired.timer
```

### Check branch traffic timers and service

Run on Branch1 and Branch2:

```bash
systemctl list-timers 'daily-lab-traffic-*' --all
systemctl status lab-traffic.service --no-pager
```

The service should be active only between 8:55 AM and 5:05 PM CT.

### View current benchmark results

```bash
sudo tail -100 /var/log/site-benchmark/*internet-iperf.log
sudo tail -30 /var/log/site-benchmark/*speedtest.log
sudo tail -30 /var/log/site-benchmark/*summary.log
```

### Check benchmark service results

```bash
systemctl show site-benchmark@baseline.service \
  -p Result -p ExecMainStatus
systemctl show site-benchmark@impaired.service \
  -p Result -p ExecMainStatus
```

### Check errors

```bash
systemctl --failed
journalctl -u site-benchmark@baseline.service -n 50 --no-pager
journalctl -u site-benchmark@impaired.service -n 50 --no-pager
journalctl -u lab-traffic.service -n 50 --no-pager
```

## 11. Manual Operations

### Run a benchmark now

This consumes internet bandwidth for the iPerf and Speedtest portions.

```bash
sudo systemctl start site-benchmark@baseline.service
systemctl show site-benchmark@baseline.service \
  -p Result -p ExecMainStatus
```

Use the impaired label when testing during an impairment window:

```bash
sudo systemctl start site-benchmark@impaired.service
```

### Start or stop branch traffic manually

```bash
sudo systemctl start lab-traffic.service
sudo systemctl stop lab-traffic.service
```

### Apply or clear direct-interface impairment manually

Run on the IGW. Example for Hub1 ISP2:

```bash
sudo /usr/local/sbin/set-site-netem.sh ens7 500ms 10%
sudo /usr/local/sbin/clear-site-netem.sh ens7
```

Example for Branch2 ISP2:

```bash
sudo /usr/local/sbin/set-branch2-isp2-netem.sh delay 500ms loss 10%
sudo /usr/local/sbin/clear-branch2-isp2-netem.sh
```

### Emergency clear of every impairment

```bash
sudo systemctl start netem-safety-clear.service
```

Then verify:

```bash
tc qdisc show dev ens7
tc qdisc show dev ens9
tc qdisc show dev ens11
tc qdisc show dev ens5
```

No active `netem` qdisc should remain after cleanup.

## 12. Validation Examples

A successful Hub2 public benchmark on October 21, 2026 produced approximately:

```text
Public iPerf reverse/download: 225 Mbps
Speedtest ping:               27.63 ms
Speedtest download:           1438.49 Mbps
Speedtest upload:             258.01 Mbps
Service Result:               success
ExecMainStatus:               0
```

Results vary with public server utilization, routing, and impairment state.

## 13. Troubleshooting

### Public iPerf reports busy or resets

This is normal for shared public servers. Confirm that the log shows attempts to the fallback endpoints:

```bash
sudo tail -100 /var/log/site-benchmark/*internet-iperf.log
```

Speedtest still runs even if every public iPerf endpoint is unavailable.

### Benchmark lock permission failure

The correct lock path is:

```text
/var/log/site-benchmark/site-benchmark.lock
```

It must not use `/run/site-benchmark.lock`, because the benchmark service runs as `ubuntu`.

Confirm the installed script:

```bash
grep 'exec 9' /usr/local/sbin/site-benchmark.sh
```

Expected output:

```bash
exec 9>"$log_dir/site-benchmark.lock"
```

### Branch2 impairment does not start

Confirm that the IFB name is `ifb-br2-isp2`, not the invalid longer name:

```bash
grep ifb-br2-isp2 /usr/local/sbin/*branch2*netem.sh
```

Then inspect:

```bash
journalctl -u netem-branch2-start.service -n 50 --no-pager
ip link show ifb-br2-isp2
tc qdisc show dev ens5
```

### Logrotate parse errors

Files copied from Windows may contain CRLF line endings. Normalize them on Linux:

```bash
sudo sed -i 's/\r$//' /etc/logrotate.d/lab-traffic
sudo sed -i 's/\r$//' /etc/logrotate.d/site-benchmark
```

Validate:

```bash
sudo logrotate -d /etc/logrotate.d/lab-traffic
sudo logrotate -d /etc/logrotate.d/site-benchmark
```

### Timer changes trigger an old job immediately

Avoid reusing an expired one-shot timer identity when changing schedules. A restarted elapsed timer can replay its associated service. Use a fresh timer name when replacing old one-shot timers, then run the safety-clear service after timer maintenance.

### Check time interpretation

```bash
date
timedatectl
systemctl list-timers --all
```

The host can remain in UTC because each timer explicitly specifies `America/Chicago`.

## 14. Local Deployment Artifacts

The following source files are maintained in `C:\Users\Administrator\Tools\PuTTY`:

```text
set-site-netem.sh
clear-site-netem.sh
set-branch2-isp2-netem.sh
clear-branch2-isp2-netem.sh
install-netem-schedule.sh
lab-traffic.sh
lab-traffic.service
install-traffic-schedule.sh
lab-traffic.logrotate
iperf3-lab.service
iperf3-branch2.service
site-benchmark.sh
site-benchmark@.service
install-benchmark-schedule.sh
site-benchmark.logrotate
```

Before installing a Windows-edited shell script on Linux, normalize line endings and validate syntax:

```bash
sed -i 's/\r$//' /tmp/script.sh
bash -n /tmp/script.sh
```

## 15. End-of-Day Checklist

1. Confirm `netem-safety-clear.service` ran successfully after 5:05 PM CT.
2. Confirm no `netem` qdisc remains on `ens7`, `ens9`, `ens11`, or the Branch2 path on `ens5`.
3. Confirm `lab-traffic.service` is inactive on both branches after 5:05 PM CT.
4. Review baseline and impaired benchmark logs for all four sites.
5. Check `systemctl --failed` on the IGW and all four test hosts.
6. Confirm the next daily timer events are listed at the expected Central times.
