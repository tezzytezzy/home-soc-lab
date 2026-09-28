Absolutely — here is the complete Markdown content in a single copy/paste-ready code block.

````
# Installing Zeek on Kali Linux: Resolving a `libc6` Dependency Conflict

**Filename:** `zeek-installation-kali-dependency-conflict.md`

**Date:** `2026-09-27`

**Status:** Resolved — Zeek repository configured, Zeek installed, and `/opt/zeek/bin` added to the user's `PATH`.

---

## 1. Overview

While setting up a defensive-security/SOC lab on Kali Linux, I attempted to install Zeek using:

```bash
sudo apt install zeek
````

 The installation initially failed because Kali's available Zeek package had an incompatible `libc6` dependency.

 The problem was eventually resolved by:

 1. Verifying the installed and candidate package versions.
2. Confirming that Kali's Zeek package was outdated/incompatible with the installed `libc6`.
3. Adding the Zeek package repository separately.
4. Refreshing APT package metadata.
5. Confirming that Zeek `9.0.0-0` became the APT candidate.
6. Installing Zeek.
7. Discovering that the `zeek` executable was installed under `/opt/zeek/bin` but was not automatically available in the shell's `PATH`.
8. Adding `/opt/zeek/bin` to the user's `PATH`.
9. Verifying the installation.

---

 # 2\. Initial Installation Attempt

 The first attempt was:

```
sudo apt install zeek
```

 APT returned:

```
Solving dependencies... Error!

Unsatisfied dependencies:
 zeek : Depends: libc6 (< 2.38) but 2.42-16 is to be installed
        Depends: libgoogle-perftools4 (>= 2.10)
        Depends: zeek-common (>= 5.1.1-0kali3) but it is not going to be installed

Error: Unable to satisfy dependencies. Reached two conflicting assignments:
   1. zeek:amd64=5.1.1-0kali3 is selected for install
   2. zeek:amd64 Depends libc6 (< 2.38)
      but none of the choices are installable:
      [no choices]
```

 The critical line was:

```
zeek : Depends: libc6 (< 2.38) but 2.42-16 is to be installed
```

---

 # 3\. Diagnose the Dependency Conflict

 Instead of immediately changing system libraries, the available package versions were inspected.

 ## Check Zeek

```
apt policy zeek
```

 Initial result:

```
zeek:
  Installed: (none)
  Candidate: 5.1.1-0kali3
  Version table:
     5.1.1-0kali3 500
        500 http://http.kali.org/kali kali-rolling/main amd64 Packages
```

 This showed that APT was selecting:

```
5.1.1-0kali3
```

 from the Kali repository.

 ## Check `libc6`

```
apt policy libc6
```

 Result:

```
libc6:
  Installed: 2.42-16
  Candidate: 2.43-4
  Version table:
     2.43-4 500
        500 http://http.kali.org/kali kali-rolling/main amd64 Packages
 *** 2.42-16 100
        100 /var/lib/dpkg/status
```

 The situation was therefore:

```
Zeek 5.1.1-0kali3
        │
        └── requires libc6 < 2.38

System:
        libc6 2.42-16
```

 The old Zeek package could not satisfy its dependency against the current system library.

 Kali's bug tracker contains a report concerning the `zeek 5.1.1-0kali3` package and dependency problems involving newer `libc6` versions.

---

 # 4\. Check the APT Repository Configuration

 The configured APT sources were inspected:

```
ls -l /etc/apt/sources.list.d/
```

 The relevant output was:

```
total 8
-rw-r--r-- 1 root root 1921 Sep 22 13:14 elastic.sources.disabled
-rw-r--r-- 1 root root  235 Sep 22 13:14 kali.sources
```

 Kali's current repository configuration uses:

```
/etc/apt/sources.list.d/kali.sources
```

 for the normal Kali repository.

 Kali recommends keeping third-party repositories separate from the primary Kali repository configuration.

---

 # 5\. Refresh APT

 Before making further changes:

```
sudo apt update
```

 The command completed successfully:

```
Hit:1 http://http.kali.org/kali kali-rolling InRelease
1179 packages can be upgraded.
```

 The message:

```
1179 packages can be upgraded
```

 was treated as a separate system-maintenance issue.

 It did **not** mean that all 1179 packages had to be upgraded to solve the Zeek dependency conflict.

---

 # 6\. Why `libc6` Was NOT Downgraded

 The obvious-looking workaround would have been to downgrade `libc6` to a version below 2.38.

 This was deliberately avoided.

 The problematic package was:

```
zeek 5.1.1-0kali3
```

 not the installed `libc6`.

 `libc6` is a fundamental Linux system library. Downgrading it simply to accommodate an outdated Zeek package could introduce additional dependency conflicts or destabilize the operating system.

 The preferred approach was:

```
Find a newer Zeek package
        ↓
compatible with the current system
```

 rather than:

```
Downgrade core system libraries
        ↓
to accommodate an old package
```

---

 # 7\. Add the Zeek Repository

 The initial `apt policy zeek` output showed that only the Kali version was available.

 Therefore, the Zeek repository was added separately.

```
echo 'deb http://download.opensuse.org/repositories/security:/zeek/Debian_Testing/ /' | \
sudo tee /etc/apt/sources.list.d/security-zeek.list
```

 The repository signing key was added:

```
curl -fsSL https://download.opensuse.org/repositories/security:zeek/Debian_13/Release.key | \
gpg --dearmor | \
sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
```

 > **Note:** The repository and signing-key paths above were the commands used during this installation. The Zeek repository is hosted through the openSUSE Build Service and publishes packages for several Debian-family targets. Repository contents and supported versions can change over time. The repository should therefore be rechecked before repeating this procedure on a future Kali installation.

---

 # 8\. Refresh APT After Adding the Zeek Repository

 APT metadata was refreshed again:

```
sudo apt update
```

 This time APT processed the additional Zeek repository.

 The important next step was to check whether APT could now see a newer Zeek package:

```
apt policy zeek
```

 The candidate changed from the Kali package:

```
5.1.1-0kali3
```

 to the newer Zeek repository package:

```
9.0.0-0
```

 The important result was effectively:

```
zeek:
  Installed: (none)
  Candidate: 9.0.0-0
```

 This confirmed that the external repository was providing a Zeek version that could be installed without requiring the old `libc6 (< 2.38)` dependency.

---

 # 9\. Install Zeek

 With the newer package available, Zeek was installed normally:

```
sudo apt install zeek
```

 APT selected the newer Zeek package rather than Kali's older `5.1.1-0kali3` package.

 The installation completed successfully.

 The package installed Zeek beneath:

```
/opt/zeek/
```

 In particular, the main executable was located at:

```
/opt/zeek/bin/zeek
```

 This installation layout is consistent with Zeek's documented binary-package installation prefix.

---

 # 10\. Initial Zeek Verification

 Immediately after installation, attempting:

```
zeek --version
```

 did not work from the normal shell because `/opt/zeek/bin` was not yet in `PATH`.

 The executable was confirmed to exist with:

```
ls -l /opt/zeek/bin/zeek
```

 or:

```
/opt/zeek/bin/zeek --version
```

 Running the binary by its full path confirmed that Zeek itself had been installed successfully.

---

 # 11\. Add `/opt/zeek/bin` to `PATH`

 For a Bash shell, the directory was added to the user's shell configuration:

```
echo 'export PATH="/opt/zeek/bin:$PATH"' >> ~/.bashrc
```

 The current shell was then updated without requiring a logout/login:

```
source ~/.bashrc
```

 Alternatively, a new terminal session can be opened.

 Verify the path:

```
echo "$PATH"
```

 The result should contain:

```
/opt/zeek/bin
```

---

 # 12\. Verify the `zeek` Command

 The shell should now be able to locate the executable:

```
which zeek
```

 Expected result:

```
/opt/zeek/bin/zeek
```

 Then:

```
zeek --version
```

 should report the installed Zeek version.

 The installation can also be verified with:

```
command -v zeek
```

 Expected:

```
/opt/zeek/bin/zeek
```

---

 # 13\. Verify Zeek's Supporting Tools

 The Zeek installation includes additional utilities. One useful check is:

```
which zeekctl
```

 Expected:

```
/opt/zeek/bin/zeekctl
```

 Then:

```
zeekctl
```

 should start ZeekControl and display its help/interactive interface.

 ZeekControl (`zeekctl`) is the management tool used for more complex Zeek deployments.

---

 # 14\. Basic Functional Test

 A simple offline test can be performed without attaching Zeek to a live network interface.

 For example:

```
zeek -N
```

 This asks Zeek to display information about loaded plugins and provides a quick indication that the executable and installation environment are functioning.

 Another useful check is:

```
zeek --help
```

 If the command displays Zeek's command-line help, the shell can successfully locate and execute the installed binary.

---

 # 15\. Confirm the Installed Package

 APT can be used to confirm which Zeek package is installed:

```
apt policy zeek
```

 The output should now show an installed version corresponding to the newer repository package, for example:

```
zeek:
  Installed: 9.0.0-0
  Candidate: 9.0.0-0
```

 The exact repository metadata and available version may change in the future.

 The important point is that the installed package should no longer be:

```
5.1.1-0kali3
```

 if the external repository was successfully selected.

---

 # 16\. Final Installation State

 The resulting installation was:

```
Kali Linux
    │
    ├── libc6 2.42-16
    │
    ├── Kali Zeek package
    │      └── 5.1.1-0kali3
    │          └── incompatible with current libc6
    │
    └── Zeek external repository
           │
           └── Zeek 9.0.0-0
                  │
                  └── /opt/zeek/bin/zeek
```

 The shell configuration was updated so that:

```
/opt/zeek/bin
```

 is included in:

```
PATH
```

 Therefore:

```
zeek
```

 and:

```
zeekctl
```

 can be invoked directly without specifying their full paths.

---

 # 17\. Final Verification Checklist

 Run the following commands to reproduce the final checks:

```
apt policy zeek
```

```
command -v zeek
```

```
zeek --version
```

```
command -v zeekctl
```

```
zeek --help
```

 Expected key results:

```
Candidate: 9.0.0-0
```

```
/opt/zeek/bin/zeek
```

 and a successful Zeek version/help response.

---

 # 18\. Important Lessons

 ### Do not downgrade `libc6` just to install an outdated package

 When a package requires an obsolete version of a core system library, first determine whether the package itself is outdated.

 In this case:

```
Old Zeek package
       ↓
requires libc6 < 2.38
       ↓
current Kali system has libc6 2.42+
```

 The better solution was to obtain a newer Zeek package.

 ### Always inspect package candidates

 Useful commands include:

```
apt policy zeek
```

 and:

```
apt policy libc6
```

 These show what APT actually knows about installed and candidate versions.

 ### Keep third-party repositories separate

 The Zeek repository was kept in its own APT source file:

```
/etc/apt/sources.list.d/security-zeek.list
```

 rather than modifying Kali's primary repository definition.

 ### Verify where binaries are installed

 A successful package installation does not necessarily mean that the executable will be immediately available as a shell command.

 In this case:

```
/opt/zeek/bin/zeek
```

 existed, but:

```
zeek
```

 was initially unavailable because `/opt/zeek/bin` was not in `PATH`.

---

 # 19\. Final Command Summary

 For future reference, the essential sequence was:

```
# 1. Check the existing package
apt policy zeek
apt policy libc6

# 2. Add the Zeek repository
echo 'deb http://download.opensuse.org/repositories/security:/zeek/Debian_Testing/ /' | \
sudo tee /etc/apt/sources.list.d/security-zeek.list

# 3. Add the repository signing key
curl -fsSL https://download.opensuse.org/repositories/security:zeek/Debian_13/Release.key | \
gpg --dearmor | \
sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

# 4. Refresh package metadata
sudo apt update

# 5. Confirm the newer Zeek candidate
apt policy zeek

# 6. Install Zeek
sudo apt install zeek

# 7. Add Zeek to PATH
echo 'export PATH="/opt/zeek/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 8. Verify
command -v zeek
zeek --version
command -v zeekctl
zeek --help
```

---

 # 20\. Conclusion

 The original installation failure was caused by an outdated Kali Zeek package:

```
zeek 5.1.1-0kali3
```

 which required:

```
libc6 < 2.38
```

 while the system had a substantially newer `libc6`.

 Rather than downgrading the system's `libc6`, the Zeek repository was added and APT was refreshed. This made the newer Zeek package available and allowed Zeek to be installed successfully.

 The final installation location was:

```
/opt/zeek/
```

 and the executable was:

```
/opt/zeek/bin/zeek
```

 Adding:

```
/opt/zeek/bin
```

 to the user's `PATH` completed the setup.

 **Final state:**

```
Zeek installed successfully
        │
        ├── Repository configured
        ├── libc6 left untouched
        ├── Zeek installed under /opt/zeek
        ├── /opt/zeek/bin added to PATH
        └── zeek --version verified
```

```

```
