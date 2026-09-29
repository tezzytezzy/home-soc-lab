# Experimental Intrusion Detection System (IDS) Lab — TShark + Suricata

A small, controlled intrusion-detection experiment demonstrating how network packets can be captured with **TShark**, analyzed offline with **Suricata**, and matched against both **default Suricata rules** and a **custom detection rule**.

The entire experiment was performed against a locally hosted HTTP server on the test machine. No external systems were scanned or attacked.

---

## 1. Objective

The objective of this project is to build and demonstrate a minimal IDS workflow:

```text
┌──────────────────────┐
│ Local HTTP Test      │
│ Server :8080         │
└──────────┬───────────┘
           │
           │ HTTP GET
           ▼
┌──────────────────────┐
│ TShark               │
│ Packet Capture       │
└──────────┬───────────┘
           │
           │ .pcapng
           ▼
┌──────────────────────┐
│ Suricata             │
│ Offline IDS Analysis │
└──────────┬───────────┘
           │
           │ Alert
           ▼
┌──────────────────────┐
│ eve.json             │
│ IDS Event            │
└──────────────────────┘
```
Technically, the completed experiment can be represented as:

```text
                  Controlled Local Lab
                  ====================

       curl
        |
        | HTTP GET
        v
  127.0.0.1:8080
        |
        v
 Python HTTP Server
        |
        | loopback traffic
        v
       lo
        |
        v
      TShark
        |
        | PCAP
        v
home-soc-alert.pcapng
        |
        | Suricata -r ... -k none
        v
     Suricata
        |
        +--> HTTP parser
        |
        +--> Custom rule SID 1000001
        |
        v
     eve.json
        |
        v
HOME-SOC TEST - HTTP GET detected
```

The experiment demonstrates:

* Capturing packets from a local interface with TShark
* Generating deliberate HTTP traffic
* Inspecting the resulting PCAP with TShark
* Configuring Suricata with its default rules
* Creating a custom Suricata signature
* Loading the custom signature alongside the default rule set
* Replaying the captured traffic through Suricata
* Confirming the resulting alert in `eve.json`

---

## 2. Systems and Software Used

### Operating System

* Linux Dell 6.19.14+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.19.14-1+kali1 (2026-05-05) x86_64 GNU/Linux

### Software
| Software       | Version                       | Purpose                                                                      |
| -------------- | ----------------------------- | ---------------------------------------------------------------------------- |
| **Suricata**   | 8.0.6                         | IDS engine; analyzes captured packets against detection rules                |
| **TShark**     | 4.6.6                         | Command-line packet capture and packet analysis                              |
| **Wireshark**  | 4.6.6                         | GUI packet-analysis tool for inspecting the captured traffic                 |
| **Python**     | 3.13.12                       | Runs the local HTTP test server                                              |
| **curl**       | 8.20.0                        | Generates the deliberate HTTP GET request                                    |
| **jq**         | 1.8.2                         | Formats and filters Suricata's JSON event logs                               |
| **Zeek**       | *not used in this experiment* | Network-security monitoring / metadata generation; not part of this IDS test |

The experiment uses only traffic generated locally by the test machine.

---

## 3. Experimental Directory Structure

The project was organized approximately as follows:

```text
home-soc-lab/
├── captures/
│   └── home-soc-alert.pcapng
│
└── suricata/
    ├── home-soc.rules
    │
    └── alert/
        ├── eve.json
        ├── fast.log
        ├── stats.log
        └── suricata.log
```

Suricata's system configuration and downloaded rule set remain under the operating system's Suricata directories:

```text
/etc/suricata/
├── suricata.yaml
├── classification.config
└── reference.config

/var/lib/suricata/rules/
├── suricata.rules
├── classification.config
└── home-soc.rules
```

The exact paths may differ on another installation.

---

## 4. Start the Local HTTP Test Server

A deliberately simple HTTP server was created using Python.

From the test-server directory:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

The server listens only on:

```text
127.0.0.1:8080
```

This ensures that the experiment remains local to the test machine.

The server can be verified with:

```bash
curl http://127.0.0.1:8080
```

The Python server should record a request similar to:

```text
127.0.0.1 - - [28/Sep/2026 19:28:34] "GET / HTTP/1.1" 200 -
```

---

## 5. Capture the Traffic with TShark

TShark was used to capture traffic from the loopback interface.

```bash
timeout 30 tshark \
    -i lo \
    -w ~/Desktop/home-soc-lab/captures/home-soc-alert.pcapng
```

The important components are:

| Option       | Purpose                           |
| ------------ | --------------------------------- |
| `timeout 30` | Stop the capture after 30 seconds |
| `-i lo`      | Capture on the loopback interface |
| `-w`         | Write packets to a PCAPNG file    |

The resulting capture is:

![https://github.com/tezzytezzy/home-soc-lab/blob/main/images/tshark-packet-capture-info.png](https://github.com/tezzytezzy/home-soc-lab/blob/main/images/tshark-packet-capture-info.png)

While TShark is capturing, generate the HTTP request from another terminal:

```bash
curl http://127.0.0.1:8080
```

The traffic now follows this path:

```text
curl
  │
  │ HTTP GET /
  ▼
127.0.0.1:8080
  │
  │ packets
  ▼
loopback interface (lo)
  │
  ▼
TShark
  │
  ▼
home-soc-alert.pcapng
```

---

## 6. Create the Custom Suricata Rule

The experiment used a custom rule called:

```text
home-soc.rules
```

The rule detects an HTTP `GET` request.

The final rule was:

```text
alert http any any -> any any (msg:"HOME-SOC TEST - HTTP GET detected"; flow:established,to_server; http.method; content:"GET"; sid:1000001; rev:1;)
```

The rule contains:

| Component                    | Meaning                                               |
| ---------------------------- | ----------------------------------------------------- |
| `alert`                      | Generate an alert                                     |
| `http`                       | Use Suricata's HTTP application-layer protocol parser |
| `any any`                    | Any source IP and source port                         |
| `->`                         | Traffic direction                                     |
| `any any`                    | Any destination IP and destination port               |
| `msg`                        | Human-readable alert message                          |
| `flow:established,to_server` | Match established traffic going toward the server     |
| `http.method`                | Inspect the HTTP method                               |
| `content:"GET"`              | Match the `GET` method                                |
| `sid:1000001`                | Unique Suricata Signature ID                          |
| `rev:1`                      | Rule revision                                         |

### Important: Keep the rule on one line

For this experiment, the Suricata signature **must be written as a single line**.

Do **not** format it like this:

```text
alert http any any -> any any (
    msg:"...";
    flow:established,to_server;
    http.method;
    content:"GET";
    sid:1000001;
    rev:1;
)
```

Suricata 8.0.6 rejected the multiline version during rule parsing and reported errors such as:

```text
Signature missing required value "sid"
```

and:

```text
no rule options
```

---

## 7. Download and Incorporate the Default Suricata Rules

The initial Suricata installation did **not** contain the downloaded default rule set.

Suricata's configuration pointed to:

```text
/var/lib/suricata/rules
```

and expected:

```text
suricata.rules
```

The default rule set was therefore downloaded before testing the custom rule.

One way to obtain the current Suricata Emerging Threats Open rules is with:

```bash
sudo suricata-update
```

This retrieves and installs the enabled rule set for Suricata.

Afterward, verify that the rule file exists:

```bash
ls -lh /var/lib/suricata/rules/
```

The directory should contain a file similar to:

```text
suricata.rules
```

Suricata's configuration can be inspected with:

```bash
grep -n -A10 -B3 "rule-files:" /etc/suricata/suricata.yaml
```

The relevant configuration used in the experiment was:

```yaml
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules
```

This tells Suricata to load:

```text
/var/lib/suricata/rules/suricata.rules
```

### Add the custom rule

The custom rule was then copied into the same rule directory:

```bash
sudo cp \
    ~/Desktop/home-soc-lab/suricata/home-soc.rules \
    /var/lib/suricata/rules/home-soc.rules
```

The configuration was changed to load both rule files:

```yaml
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules
  - home-soc.rules
```

The resulting rule-loading relationship is:

```text
/etc/suricata/suricata.yaml
              │
              │ default-rule-path
              ▼
/var/lib/suricata/rules/
              │
       ┌──────┴────────┐
       ▼               ▼
suricata.rules   home-soc.rules
       │               │
       │ default       │ custom
       │ rules         │ HTTP GET
       └───────┬───────┘
               ▼
          Suricata IDS
```

### Configuration validation

Before analyzing traffic, validate the configuration:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

A successful test should indicate that Suricata was able to load the configuration and signatures without parsing errors.

---

## 8. Permission Consideration When Using Downloaded Templates

When obtaining a Suricata configuration/rule template from another location, the source file may reside in a directory that the Suricata process cannot read.

For this experiment, the template was copied into `/tmp/` before being used.

For example:

```bash
cp /path/to/template.yml /tmp/suricata-template.yml
```

Then inspect it:

```bash
less /tmp/suricata-template.yml
```

If the template is being incorporated into the Suricata configuration, copy or merge the required configuration into:

```text
/etc/suricata/suricata.yaml
```

using appropriate privileges.

The important principle is:

```text
Downloaded template
       │
       ▼
/tmp/
       │
       ▼
inspect / modify
       │
       ▼
/etc/suricata/
```

This avoids confusing a **file-access permission problem** with a Suricata configuration or rule-parsing problem.

---

## 9. Inspect the Captured HTTP Traffic

Before sending the PCAP to Suricata, verify that the HTTP request was actually captured.

![https://github.com/tezzytezzy/home-soc-lab/blob/main/images/suricata-eve.json-creation.png](https://github.com/tezzytezzy/home-soc-lab/blob/main/images/tshark-http-packet-verification.png)

This confirms that the PCAP contains the traffic required by the custom rule.

---

## 10. Analyze the PCAP with Suricata

Suricata can process the captured PCAP offline:

![https://github.com/tezzytezzy/home-soc-lab/blob/main/images/ids-alert-in-eve.json.png](https://github.com/tezzytezzy/home-soc-lab/blob/main/images/suricata-eve.json-creation.png)

### Why `-k none`?

The local loopback capture can contain packets with checksums that are not valid when examined later from the saved PCAP.

Suricata may therefore report:

```text
pcap: 1/1th of packets have an invalid checksum
```

The experiment demonstrated that disabling checksum validation with:

```text
-k none
```

allows Suricata to analyze the locally captured traffic and trigger the custom alert.

This option is particularly relevant to this **local experimental loopback capture**. It should not be treated as a general recommendation to disable checksum validation for arbitrary captures.

---

## 11. Verify the IDS Alert

Suricata writes JSON-formatted events to:

```text
~/Desktop/home-soc-lab/suricata/alert/eve.json
```
Search for the custom signature:
![https://github.com/tezzytezzy/home-soc-lab/blob/main/images/ids-alert-in-eve.json.png](https://github.com/tezzytezzy/home-soc-lab/blob/main/images/ids-alert-in-eve.json.png)

This confirms that:

1. The HTTP request was captured.
2. Suricata successfully decoded the HTTP traffic.
3. The custom signature matched the request.
4. Suricata generated an IDS alert.
5. The alert was recorded in `eve.json`.

---

## 12. Where Zeek Fits

**Zeek is not required for this particular experiment.**

Suricata and Zeek can both operate as network-security monitoring tools, but they serve somewhat different purposes.

For this project:

```text
TShark
  │
  │ raw packet capture
  ▼
PCAP
  │
  ▼
Suricata
  │
  │ signature-based detection
  ▼
IDS alert
```

Zeek would fit into a broader network-monitoring pipeline:

```text
                    Network Traffic
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
          Suricata                  Zeek
              │                       │
              │ Alerts                │ Network metadata
              │                       │
              ▼                       ▼
          eve.json               Zeek logs
```

Suricata is particularly appropriate for this experiment because the objective is to demonstrate a **custom IDS signature triggering an alert**.

Zeek becomes useful in a future experiment when the objective changes from:

> "Did this traffic match my detection rule?"

to questions such as:

> "Who communicated with whom?"

> "Which protocols were used?"

> "What DNS activity occurred?"

> "What HTTP connections happened?"

> "What network behavior occurred over time?"

Therefore, Zeek is intentionally **outside the scope of this repository's main experiment**.

---

## 13. Results

The experiment successfully demonstrated the complete detection pipeline:

```text
HTTP GET generated
        ↓
Packet captured by TShark
        ↓
PCAP verified
        ↓
Suricata loaded default rules
        ↓
Custom rule loaded
        ↓
PCAP replayed through Suricata
        ↓
Custom signature matched
        ↓
Alert written to eve.json
```

The final custom signature used:

```text
SID: 1000001
```

and generated:

```text
HOME-SOC TEST - HTTP GET detected
```

The experiment therefore provides a reproducible demonstration of a minimal **packet-capture → IDS-analysis → alert** workflow.

---

## 14. Security and Privacy Notes

This experiment was deliberately restricted to the local machine.

The HTTP server was bound to:

```text
127.0.0.1
```

and therefore was not intended to expose the test service to other machines.

Before publishing PCAPs, logs, screenshots, or terminal output to GitHub, inspect them for:

* Public IP addresses
* Private network addresses that you do not want to publish
* Hostnames
* Usernames
* MAC addresses
* Email addresses
* Authentication tokens
* Cookies
* API keys
* File paths containing personal information
* Browser or application identifiers

For this repository, the experiment should remain focused on **authorized, locally generated traffic**.

---

## 15. Key Lessons

This experiment demonstrates several fundamental IDS concepts:

### Packet capture is separate from detection

TShark captures the traffic:

```text
TShark → PCAP
```

while Suricata analyzes the captured traffic:

```text
PCAP → Suricata → Alert
```

### Rules determine what Suricata detects

The custom signature defines the specific behavior we want to identify.

### A PCAP can be replayed

Offline analysis makes it possible to repeatedly test detection logic against the same traffic without generating new network activity.

### Alerts contain useful context

The Suricata JSON event provides information such as:

* Source/destination
* Ports
* Protocol
* Signature ID
* Signature message
* HTTP method
* URL
* Application protocol
* Severity

### Detection engineering is iterative

A useful IDS workflow is:

```text
Generate known traffic
        ↓
Capture it
        ↓
Inspect the packets
        ↓
Write a detection rule
        ↓
Test the rule
        ↓
Investigate the alert
        ↓
Refine the rule
```

That feedback loop is the central idea demonstrated by this project.
