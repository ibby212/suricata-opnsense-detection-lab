# Network Attack & Detection Lab: OPNsense + Suricata IDS

A two-person attack/detection lab, built as part of a four-student EPITA course project, in which I ran the **defender** side. A teammate launched a live, multi-stage web attack against a vulnerable **DVWA** target sitting behind my **OPNsense** firewall over a Tailscale link. I ran the **Suricata** intrusion-detection sensor, wrote the detection rules, tuned them, and reconstructed the incident from the logs.

## My role

Defender. I hosted the OPNsense firewall, the DVWA target VM and the Suricata sensor, and read every alert from the Suricata console and `eve.json`. A teammate played the attacker and ran the attacks live. I did not script them in advance, which is what made the detection and the incident timeline realistic.

## What I built

### Five custom Suricata rules

| SID | Detects | Attack stage |
|-----|---------|--------------|
| `9000011` | `uid=`/`gid=` command output in the server response | Command injection → RCE |
| `9000010` | `c=` command parameter on an uploaded script under `/hackable/uploads/` | Web shell |
| `9000012` | `/etc/passwd` path in the request | Local file inclusion |
| `9000040` | `uid=`/`gid=` shell output over raw TCP | Bind shell |
| `9000030` | 20+ DNS queries to one domain in 10 seconds | DNS tunnelling / exfiltration |

The rules are result-based: they match the outcome of an attack (the `uid=` output, the `/etc/passwd` contents) rather than one specific payload. A command injection can be written many ways, but a successful one returns the same shaped output, so matching the result catches more variants than signaturing each input.

### False-positive tuning

Rule `9000010` first matched `c=` anywhere in the URL, so benign query parameters (`src=`, `desc=`, `loc=`) all fired it: five false positives against two minutes of normal browsing, counted from matches in `eve.json`. I tightened the rule to also require the `/hackable/uploads/` path. That dropped it to zero false positives while the real web-shell request still fired.

### Capstone: incident timeline

My teammate ran the full chain unannounced (command injection → web shell → bind shell → DNS exfiltration) and I rebuilt the incident timeline from `eve.json`: every stage in order, each with its SID, source and timestamp.

## Key lessons

- Egress filtering on its own is not enough. A reverse shell blocked on port 4444 moved to port 443 and left over standard encrypted web traffic (HTTPS), which can't be blocked without breaking normal browsing. Catching that needs traffic inspection, not a port rule.
- DNS tunnelling survives network isolation. A tunnel query only travels to the resolver (the gateway), which every host is always allowed to reach, so it still fires even when the two machines can't talk to each other directly.
- Encrypted exfil defeats content inspection. An HTTPS POST shows Suricata only the TLS metadata, not the payload, so it has to be caught by volume and destination anomaly instead of a signature.

## Scope and caveats

- The lab ran Suricata in IDS / alert mode. The promote-to-drop (IPS) step is documented but not enforced live: IPS/netmap on the virtio LAN interface overflowed under load and destabilised the VMs, so the rule change is shown rather than run in blocking mode.
- Attacker requests are attributed to `192.168.1.1` in the alerts. The Tailscale subnet router source-NATs the attacker's traffic to its own LAN address before it reaches the sensor.

## Contents

- **[`Advanced-Attack-Detection-Lab.pdf`](Advanced-Attack-Detection-Lab.pdf)**: the full lab report, with every attack command, the Suricata alert it produced, the custom rule, the capstone timeline and the reflection questions, plus screenshots throughout.
- **[`rules/lab-custom.rules`](rules/lab-custom.rules)**: the five custom Suricata rules.
