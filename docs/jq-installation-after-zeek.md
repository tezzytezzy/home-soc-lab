# Installing `jq` Safely After Zeek on Kali Linux 2026.2

## Overview

This document records the installation of `jq` as part of my Home SOC lab.

The installation was performed **after installing Zeek from the official Zeek distribution**, rather than installing Zeek from Kali's package repository.

This distinction is important because the Kali-packaged version of Zeek had a `libc6` compatibility issue with my Kali Linux 2026.2 environment. Because of this, I deliberately avoided performing a system-wide `full-upgrade` merely to install another tool.

Instead, I used APT's simulation functionality to inspect exactly what would change before allowing the installation.

The goal was to make a **small, controlled package change without modifying the existing Zeek installation or upgrading unrelated system packages**.

---

## Why `jq`?

`jq` is a command-line JSON processor.

It will be used in this Home SOC project to analyze structured security logs, particularly:

* Suricata EVE JSON output
* JSON-formatted Zeek logs
* Other structured security telemetry

For example, Suricata can produce events in JSON format:

```text
eve.json
```

`jq` allows those events to be filtered and transformed from large JSON records into information that is easier for a security analyst to investigate.

Example:

```bash
jq 'select(.event_type=="alert")' eve.json
```

This allows me to focus specifically on Suricata alert events.

---

# Why I Did Not Run `full-upgrade`

A previous issue was encountered when attempting to use Kali's packaged version of Zeek.

The relevant dependency relationship was:

```text
Kali-packaged Zeek
        │
        └── requires an older libc6 version
                         │
                         ▼
              Kali Linux 2026.2
              has newer libc6
```

My system currently has:

```text
libc6 2.42-16
```

Because of this compatibility issue, I installed Zeek using the official Zeek distribution instead of attempting to force the Kali package to work by changing the system's `libc6` version.

This led to an important system-administration principle for this project:

> **Do not perform a system-wide upgrade simply because one additional tool needs to be installed.**

A full upgrade could potentially modify hundreds or thousands of packages and introduce unnecessary changes to a working security lab.

At the time of installing `jq`, APT reported:

```text
Not Upgrading: 1194
```

Therefore, there was no reason to perform a system-wide upgrade just to obtain `jq`.

---

# Step 1 — Verify the Current `libc6` Version

Before installing `jq`, I checked the currently installed `libc6` version:

```bash
dpkg-query -W -f='${Package} ${Version}\n' libc6
```

Output:

```text
libc6 2.42-16
```

### Why?

`libc6` is a fundamental system library.

Because the previous Zeek installation issue involved `libc6`, I wanted to verify the current version before making another package change.

This provides a baseline for the system.

---

# Step 2 — Simulate the `jq` Installation

Instead of immediately installing `jq`, I asked APT to simulate the installation:

```bash
sudo apt install --simulate jq
```

The simulation produced:

```text
Installing:
  jq

Installing dependencies:
  libjq1 libonig5

Summary:
  Upgrading: 0, Installing: 3, Removing: 0, Not Upgrading: 1194
```

APT further reported:

```text
Inst libonig5 (6.9.10-1+b1 kali-rolling [amd64])
Inst libjq1 (1.8.2-1 kali-rolling [amd64])
Inst jq (1.8.2-1 kali-rolling [amd64])
```

---

# Step 3 — Analyze the Proposed Transaction

The most important part of the output was:

```text
Upgrading: 0
Installing: 3
Removing: 0
```

This means APT planned to:

```text
Install:
    jq
    libjq1
    libonig5

Upgrade:
    Nothing

Remove:
    Nothing
```

The output also showed:

```text
Not Upgrading: 1194
```

This was desirable.

It demonstrated that installing `jq` did **not** require upgrading the large number of packages currently held at their existing versions.

Most importantly, the simulation did not propose removing Zeek or modifying `libc6`.

---

# Step 4 — Install `jq`

After reviewing the simulated transaction, I proceeded with the actual installation:

```bash
sudo apt install jq
```

The installation was intentionally performed without:

```bash
sudo apt full-upgrade
```

and without:

```bash
sudo apt upgrade
```

The objective was to make the smallest necessary system change.

---

# Step 5 — Verify the Installation

After installation, I checked the installed version:

```bash
jq --version
```

Output:

```text
jq-1.8.2
```

This confirmed that `jq` was successfully installed and available from the command line.

---

# Package Change Summary

The final change was:

| Package    | Action               | Version                        |
| ---------- | -------------------- | ------------------------------ |
| `jq`       | Installed            | `1.8.2-1`                      |
| `libjq1`   | Installed dependency | `1.8.2-1`                      |
| `libonig5` | Installed dependency | `6.9.10-1+b1`                  |
| `libc6`    | **Not changed**      | `2.42-16`                      |
| Zeek       | **Not changed**      | Existing official installation |

The important observation is that the `jq` installation was isolated from the existing Zeek installation.

---

# Why This Matters for a Security Lab

Installing security tools is not only about getting the tools to work.

A security lab also needs to be:

* reproducible
* stable
* documented
* change-controlled
* understandable

Blindly running:

```bash
sudo apt full-upgrade -y
```

would introduce a large number of unrelated system changes.

Instead, this procedure followed:

```text
Identify requirement
        ↓
Check current system state
        ↓
Simulate package transaction
        ↓
Review proposed changes
        ↓
Approve minimal change
        ↓
Install
        ↓
Verify
        ↓
Document
```

This approach reduces the chance of unintentionally changing the environment that other security tools depend on.

---

# Security Engineering Principle

The key lesson from this installation is:

> **Make the smallest change necessary to accomplish the objective, and verify the proposed change before applying it.**

In this case, the objective was simply:

```text
Install jq
```

It was **not**:

```text
Upgrade Kali
```

Those are two different operations.

By using:

```bash
sudo apt install --simulate jq
```

first, I was able to inspect APT's proposed transaction before modifying the system.

---

# Home SOC Use Case

`jq` will be used later in this project to analyze Suricata and other JSON-based security telemetry.

For example:

```bash
jq 'select(.event_type=="alert")' eve.json
```

A more focused investigation could extract specific fields:

```bash
jq -r '
  select(.event_type=="alert") |
  [
    .timestamp,
    .src_ip,
    .dest_ip,
    .alert.signature
  ] |
  @tsv
' eve.json
```

This converts structured security telemetry into analyst-friendly output.

The eventual workflow will be:

```text
Network Traffic
       ↓
   Suricata
       ↓
   EVE JSON
       ↓
      jq
       ↓
Relevant Security Events
       ↓
    Investigation
```

---

# Lessons Learned

### 1. Package installation and system upgrading are different operations

Installing one package does not require upgrading the entire operating system.

### 2. Simulate before modifying

APT provides a useful mechanism for previewing a transaction:

```bash
sudo apt install --simulate <package>
```

This should be particularly useful when working on a security lab containing multiple tools with potentially complex dependencies.

### 3. Dependency conflicts require context

The previous Zeek/libc6 problem demonstrated that installing a security tool can involve system-library compatibility.

Therefore, package management should be treated as part of the lab's engineering process rather than as an afterthought.

### 4. Preserve working components

Because Zeek was already working through its official installation method, I avoided unnecessary changes that could destabilize it.

### 5. Verify after installation

The installation was not considered complete until:

```bash
jq --version
```

confirmed that the expected version was available.

---

# Final State

The Home SOC environment now contains:

```text
Kali Linux 2026.2
│
├── Zeek
│   └── Installed separately from Kali's problematic package
│
├── jq 1.8.2
│   └── Installed through Kali APT
│
└── libc6 2.42-16
    └── Unchanged
```

This provides the foundation for analyzing structured security telemetry without unnecessarily modifying the rest of the operating system.

---

## Reproduction Checklist

For another installation of this lab, the relevant procedure is:

```bash
# Check libc6
dpkg-query -W -f='${Package} ${Version}\n' libc6

# Preview the jq installation
sudo apt install --simulate jq

# Review:
# - Upgrading
# - Installing
# - Removing
# - Not Upgrading

# If the transaction is acceptable:
sudo apt install jq

# Verify
jq --version
```

The critical step is **reviewing the simulation before executing the actual installation**.
