# TShark Non-Root Packet Capture and Privilege Investigation

## Overview

While building the Home SOC lab, I encountered an unexpected permission error when attempting to have TShark write a packet capture directly into the project's `captures` directory.

The initial command was:

```bash
sudo timeout 30 tshark -i 1 -w /home/<USER>/Desktop/home-soc-lab/captures/tshark-baseline.pcapng
```

TShark successfully identified the wireless interface:

```text
Capturing on 'wlan0'
```

but then failed with:

```text
tshark: The file to which the capture would be saved (...) could not be opened: Permission denied.
```

Because the command was being executed with `sudo`, the initial assumption was that there might be a filesystem permission problem.

Rather than changing permissions indiscriminately, a series of controlled tests was performed to determine the actual cause.

The investigation ultimately demonstrated that **TShark could successfully capture packets and write the capture file when executed as the normal user, without `sudo`**.

The Home SOC lab therefore uses the non-root invocation.

---

# 1. Initial Problem

The original command was:

```bash
sudo timeout 30 tshark -i 1 -w /home/<USER>/Desktop/home-soc-lab/captures/tshark-baseline.pcapng
```

TShark reported:

```text
Running as user "root" and group "root". This could be dangerous.
Capturing on 'wlan0'
tshark: The file to which the capture would be saved (...) could not be opened: Permission denied.
```

There were two separate messages here.

### Root warning

```text
Running as user "root" and group "root". This could be dangerous.
```

This was a warning about running TShark with elevated privileges.

It was not, by itself, the cause of the failure.

### Permission error

```text
could not be opened: Permission denied
```

This was the actual failure requiring investigation.

---

# 2. First Hypothesis: Directory Permissions

The first step was to inspect the project directory:

```bash
ls -ld ~/Desktop/home-soc-lab
```

The directory was owned by the normal user and had normal read/write/execute permissions.

The capture directory was then inspected:

```bash
ls -ld ~/Desktop/home-soc-lab/captures
```

The result showed:

```text
drwxrwxr-x <USER> <USER> ...
```

This indicated that the normal user owned the directory and had read, write, and execute permissions.

The contents were also inspected:

```bash
ls -la ~/Desktop/home-soc-lab/captures
```

An existing PCAPNG capture was present.

This was the first indication that the problem was probably not a simple inability to write to the directory.

---

# 3. Testing Root's Ability to Write

Instead of changing permissions, a controlled test was performed.

```bash
sudo touch /home/<USER>/Desktop/home-soc-lab/captures/test-root-write
```

The command completed successfully.

This demonstrated that root could create a file in the target directory.

Therefore:

```text
root → captures directory
       WRITE: SUCCESS
```

The problem was not simply that root lacked filesystem write permission.

The diagnostic file was temporary and was later removed.

---

# 4. Checking ACLs

The directory's Access Control List was inspected:

```bash
getfacl /home/<USER>/Desktop/home-soc-lab/captures
```

The relevant output showed:

```text
# owner: <USER>
# group: <USER>
user::rwx
group::rwx
other::r-x
```

No unexpected ACL entries were present.

This provided additional evidence that filesystem ACLs were not responsible for the TShark failure.

---

# 5. Checking the Filesystem

The filesystem containing the capture directory was examined:

```bash
findmnt -T /home/<USER>/Desktop/home-soc-lab/captures
```

The directory was located on the system's normal `ext4` root filesystem.

The filesystem was mounted read/write:

```text
rw
```

This ruled out a read-only filesystem as the explanation.

---

# 6. Checking AppArmor

Because Linux security mechanisms can sometimes restrict application behavior independently of traditional Unix permissions, AppArmor was also investigated.

The AppArmor status was checked with:

```bash
sudo aa-status
```

AppArmor was loaded, but no TShark-specific profile was identified.

Kernel messages were also checked:

```bash
sudo dmesg | grep -i -E 'apparmor|denied' | tail -30
```

No relevant AppArmor denial associated with TShark was found.

Therefore, there was insufficient evidence to identify AppArmor as the cause.

No AppArmor profiles were modified or disabled.

---

# 7. Testing TShark in `/tmp`

The next test isolated TShark from the project directory.

TShark was instructed to capture only ten packets and write the result to `/tmp`:

```bash
sudo tshark -i 1 -c 10 -w /tmp/test-tshark.pcapng
```

The capture completed successfully:

```text
Capturing on 'wlan0'
10
```

The resulting file was verified:

```bash
ls -lh /tmp/test-tshark.pcapng
```

The file existed and contained captured packet data.

This established that:

```text
TShark capture functionality       SUCCESS
PCAPNG file creation               SUCCESS
Writing to /tmp                    SUCCESS
```

---

# 8. Testing a Copy into the Project Directory

The successful capture was copied into the Home SOC project's capture directory:

```bash
sudo cp /tmp/test-tshark.pcapng \
    ~/Desktop/home-soc-lab/captures/test-copy.pcapng
```

This succeeded.

The resulting file was verified:

```bash
ls -lh ~/Desktop/home-soc-lab/captures/test-copy.pcapng
```

The file was present and owned by the normal user after the test workflow.

This established:

```text
root → project directory                 SUCCESS
copy PCAPNG → project directory          SUCCESS
```

Therefore, the project directory itself was not preventing PCAPNG files from being created.

---

# 9. Checking the TShark Installation

The installed TShark version was checked:

```bash
tshark --version
```

The system was running:

```text
TShark (Wireshark) 4.6.6
```

The executable location was checked:

```bash
which tshark
```

Result:

```text
/usr/bin/tshark
```

The executable permissions were inspected:

```bash
ls -l "$(which tshark)"
```

The executable was owned by root and was executable by normal users.

Finally, Linux file capabilities were checked:

```bash
getcap "$(which tshark)"
```

No capabilities were directly reported on the TShark executable.

This did not by itself explain how the normal user could capture packets, but it demonstrated that the necessary capture privileges were not simply being attached to `/usr/bin/tshark` through file capabilities.

---

# 10. The Decisive Test

The most important test was to remove `sudo` entirely.

The following command was executed as the normal user:

```bash
tshark -i 1 -c 10 \
    -w ~/Desktop/home-soc-lab/captures/user-test.pcapng
```

The result was:

```text
Capturing on 'wlan0'
10
```

The capture completed successfully.

This was the decisive observation.

The system demonstrated that the normal user could:

1. Access the `wlan0` capture interface.
2. Capture packets.
3. Create a PCAPNG file.
4. Write the file directly into the Home SOC project's `captures` directory.

---

# 11. Comparison of the Two Approaches

The investigation produced the following results:

| Test                                 | Result |
| ------------------------------------ | -----: |
| Root can create a file in `captures` |   PASS |
| ACL configuration is normal          |   PASS |
| Filesystem is mounted read/write     |   PASS |
| AppArmor denial identified           |     NO |
| `sudo tshark` → `/tmp`               |   PASS |
| Copy PCAPNG → `captures`             |   PASS |
| Normal-user TShark → `captures`      |   PASS |
| `sudo tshark` → `captures`           |   FAIL |

The important distinction was:

```text
sudo tshark
     │
     └── Permission denied when opening target capture
```

versus:

```text
tshark
     │
     └── Capture succeeds
```

---

# 12. Final Decision

The Home SOC lab will use TShark **without `sudo`**.

For example:

```bash
tshark -i 1 -c 10 \
    -w ~/Desktop/home-soc-lab/captures/test.pcapng
```

For a timed baseline capture:

```bash
timeout 30 tshark -i 1 \
    -w ~/Desktop/home-soc-lab/captures/tshark-baseline.pcapng
```

The exact username and other system-specific information are omitted from this public documentation.

---

# 13. Why the Non-Root Approach Is Preferred

The successful non-root capture is not merely a workaround.

It also follows the **principle of least privilege**.

A security principle commonly applied to defensive systems is:

> Give a process only the privileges it actually needs.

In this case, the Home SOC lab needs TShark to:

```text
Capture network traffic
        ↓
Write a PCAPNG file
        ↓
Allow subsequent analysis
```

There is no demonstrated requirement for the entire TShark process to run as `root`.

Running the capture as the normal user therefore reduces unnecessary privilege.

This also eliminates the warning:

```text
Running as user "root" and group "root".
This could be dangerous.
```

---

# 14. Security Engineering Lesson

A useful lesson from this troubleshooting process is:

> **Do not immediately weaken security controls when a permission error occurs.**

A less careful troubleshooting approach might have been:

```bash
sudo chmod -R 777 ~/Desktop/home-soc-lab
```

This was deliberately avoided.

Such a command would change permissions broadly without establishing the actual cause of the problem.

Instead, the investigation followed a controlled process:

```text
Permission error
       │
       ▼
Check directory permissions
       │
       ▼
Check ownership / ACL
       │
       ▼
Check filesystem
       │
       ▼
Check security controls
       │
       ▼
Test TShark in /tmp
       │
       ▼
Test TShark as normal user
       │
       ▼
Successful capture
       │
       ▼
Use least-privileged workflow
```

This approach minimizes unnecessary system changes and produces evidence for each troubleshooting decision.

---

# 15. Home SOC Capture Architecture

The resulting workflow is:

```text
                    NETWORK
                       │
                       ▼
                     wlan0
                       │
                       ▼
                  ┌─────────┐
                  │ TShark  │
                  │  User   │
                  └────┬────┘
                       │
                       ▼
                    PCAPNG
                       │
                       ▼
       home-soc-lab/captures/
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Wireshark           Zeek
          Analysis            Analysis
```

The capture process does not require the entire analysis workflow to run with root privileges.

---

# 16. Troubleshooting Commands Reference

### Check interface configuration

```bash
ip addr
```

### Check routing

```bash
ip route
```

### Check directory permissions

```bash
ls -ld ~/Desktop/home-soc-lab/captures
```

### Check ACLs

```bash
getfacl ~/Desktop/home-soc-lab/captures
```

### Check filesystem

```bash
findmnt -T ~/Desktop/home-soc-lab/captures
```

### Check TShark version

```bash
tshark --version
```

### Check TShark location

```bash
which tshark
```

### Check Linux capabilities

```bash
getcap "$(which tshark)"
```

### Test a small capture

```bash
tshark -i 1 -c 10 \
    -w ~/Desktop/home-soc-lab/captures/test.pcapng
```

---

# 17. Lessons Learned

### 1. Test before changing permissions

A permission error does not automatically mean that permissions should be changed.

### 2. Isolate variables

Testing `/tmp` helped distinguish TShark's capture/write functionality from the project directory.

### 3. Test with and without elevated privileges

Comparing:

```bash
sudo tshark ...
```

with:

```bash
tshark ...
```

provided the decisive evidence.

### 4. Prefer least privilege

If a normal user can perform the required operation, there is no reason to run the entire process as root solely out of habit.

### 5. Document evidence, not assumptions

The investigation did not require determining every internal reason for the `sudo tshark` behavior.

What matters operationally is the reproducible observation:

```text
sudo tshark → target directory: FAIL
normal-user tshark → target directory: SUCCESS
```

That evidence is sufficient to select the working, lower-privilege workflow for this lab.

---

# Conclusion

The initial TShark error appeared to be a conventional filesystem permission problem:

```text
Permission denied
```

However, controlled testing demonstrated that:

* The target directory was writable.
* Root could create files there.
* The filesystem was mounted read/write.
* No relevant AppArmor denial was identified.
* TShark could successfully create PCAPNG files.
* TShark could capture packets from `wlan0`.
* The normal user could perform the complete capture operation successfully.

The Home SOC therefore uses:

```bash
tshark -i 1 -w <capture-file>
```

rather than:

```bash
sudo tshark -i 1 -w <capture-file>
```

This provides a working packet-capture workflow while avoiding unnecessary root privileges.

The investigation itself is part of the Home SOC learning objective: **observe, test, isolate, document, and make the smallest justified change.**
