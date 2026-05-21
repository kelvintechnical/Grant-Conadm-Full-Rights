# Lab 22-1b: Grant Full Sudo Rights — `conadm`

**RHCSA EX200 Lab** | Series: Container Management (Lab 22)  
**Prerequisite:** Lab 22-1a complete (`conadm` user exists on `server30`)  
**Time Estimate:** ~5 minutes

---

## 🎯 Objective

Grant the `conadm` user full `sudo` privileges on `server30` by adding them to the `wheel` group — RHEL's designated admin group.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| Add user to wheel group | `usermod -aG wheel <username>` |
| Verify group membership | `id <username>` |
| Confirm sudoers config | `sudo grep wheel /etc/sudoers` |
| Switch to target user | `su - <username>` |
| Test sudo elevation | `sudo whoami` |
| Edit sudoers safely (alternative) | `visudo` |

---

## 🧠 Concept: Primary vs. Supplementary Groups

Before running commands, understand what you're actually doing here.

When a user is created, Linux assigns two types of group membership:

| Type | What it is | Example |
|---|---|---|
| **Primary group** | Auto-created, same name as the user | `conadm` (gid=1001) |
| **Supplementary groups** | Additional groups granting extra permissions | `wheel` (gid=10) |

Every file a user creates is owned by their **primary group**. **Supplementary groups** layer on additional permissions — in this case, the ability to invoke `sudo`.

> **Why does this matter?** Without `wheel` membership, `conadm` can log in but cannot run any administrative commands. With `wheel`, the user can prefix any command with `sudo` to run it as root.

---

## 🧠 Concept: What Is `wheel`?

`wheel` is a **system group** (GID 10) that RHEL/CentOS designates as the sudo-allowed group. Membership in `wheel` = permission to use `sudo`.

> **Historical note:** The name comes from 1970s Unix slang — *"big wheel"* referred to someone with authority. The term carried forward into Linux as the name for the privileged admin group.

Other distributions use different group names for the same purpose:

| Distro | Sudo group |
|---|---|
| RHEL / CentOS / Fedora / Amazon Linux | `wheel` |
| Ubuntu / Debian | `sudo` |
| Arch Linux | `wheel` |

> **RHCSA Exam note:** On RHEL, `wheel` sudo access is **enabled by default** in `/etc/sudoers`. You just need to add the user to the group — no file editing required.

---

## 🔧 Steps

### Step 1 — Confirm the starting state (pre-sudo)

Before making changes, verify the current state from the terminal session shown below:

```bash
id conadm ; grep conadm /etc/passwd
```

**Actual output:**

```
uid=1001(conadm) gid=1001(conadm) groups=1001(conadm)
conadm:x:1001:1001::/home/conadm:/bin/bash
```

#### Output explained

| Part | Meaning |
|---|---|
| `uid=1001(conadm)` | `conadm` is UID 1001 — the first standard user after `ec2-user` (1000). |
| `gid=1001(conadm)` | Primary group is `conadm` — Linux's User Private Group scheme. |
| `groups=1001(conadm)` | Only one group listed — `wheel` is **not yet present**. This confirms sudo is not yet granted. |
| `/bin/bash` | Default shell assigned at user creation. |

> **Baseline check habit:** Always run `id <user>` before and after modifying group membership. It makes changes visible and gives you a before/after record — useful for both labs and production systems.

---

### Step 2 — Attempt to view home directory (without sudo)

```bash
ls -la /home/conadm
```

**Output:**

```
ls: cannot open directory '/home/conadm': Permission denied
```

#### Why did this fail?

`useradd` sets home directory permissions to `700` (`drwx------`) — **owner-only**. You're logged in as `ec2-user`, not `conadm` or root. The kernel rejects the attempt.

**Correct command:**

```bash
sudo ls -la /home/conadm
```

**Output:**

```
total 12
drwx------. 2 conadm conadm  62 May 21 14:45 .
drwxr-xr-x. 4 root   root    36 May 21 14:45 ..
-rw-r--r--. 1 conadm conadm  18 Oct 29  2024 .bash_logout
-rw-r--r--. 1 conadm conadm 144 Oct 29  2024 .bash_profile
-rw-r--r--. 1 conadm conadm 522 Oct 29  2024 .bashrc
```

#### Output explained

| Entry | Permissions | Meaning |
|---|---|---|
| `.` | `drwx------` | The home directory itself. Only `conadm` (owner) and root can enter. |
| `..` | `drwxr-xr-x` | The `/home` parent — world-traversable, owned by root. |
| `.bash_logout` | `-rw-r--r--` | Executes on logout from a login shell. Clears the terminal by default. |
| `.bash_profile` | `-rw-r--r--` | Runs at login shell startup — sets `$PATH` and environment variables. |
| `.bashrc` | `-rw-r--r--` | Runs on every interactive shell. Defines aliases and prompt customization. |

> These files were copied from `/etc/skel/` by `useradd` when the account was created. They give the new user a functional shell environment out of the box.

---

### Step 3 — Add `conadm` to the `wheel` group

```bash
sudo usermod -aG wheel conadm
```

**Expected output:** *(none — silence means success)*

#### Command breakdown

| Part | Meaning |
|---|---|
| `usermod` | Modifies an existing user account |
| `-a` | **Append** — adds the new group without removing existing ones |
| `-G wheel` | Specifies `wheel` as the supplementary group to add |
| `conadm` | The target user |

> **Critical distinction — `-aG` vs `-G`:**
> - `-aG wheel` → **appends** `wheel` to existing groups ✅
> - `-G wheel` alone → **replaces** all supplementary groups with only `wheel` ❌
>
> Forgetting `-a` is one of the most common mistakes in user management. It can silently strip a user from every other group they belong to.

---

### Step 4 — Verify the group was added

```bash
id conadm
```

**Actual output:**

```
uid=1001(conadm) gid=1001(conadm) groups=1001(conadm),10(wheel)
```

#### Output explained

| Field | Before | After |
|---|---|---|
| `uid` | `1001(conadm)` | `1001(conadm)` — unchanged |
| `gid` | `1001(conadm)` | `1001(conadm)` — unchanged |
| `groups` | `1001(conadm)` | `1001(conadm),10(wheel)` ✅ |

> **Why is wheel GID 10?** Groups created during OS installation receive low GIDs (under 1000). `wheel` is a system group — it was configured when the OS was installed, so it got GID 10. User-created groups like `conadm` start at 1000+.

You can verify the full group entry:

```bash
grep wheel /etc/group
```

**Expected output:**

```
wheel:x:10:ec2-user,conadm
```

| Field | Meaning |
|---|---|
| `wheel` | Group name |
| `x` | Password (rarely used for groups) |
| `10` | GID — assigned at OS installation |
| `ec2-user,conadm` | All members of this group |

---

### Step 5 — Confirm `wheel` is enabled in `/etc/sudoers`

```bash
sudo grep wheel /etc/sudoers
```

**Actual output:**

```
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```

#### Output explained line by line

| Line | Meaning |
|---|---|
| `## Allows people in group wheel...` | Comment block — documentation only, no effect. |
| `%wheel  ALL=(ALL)  ALL` | **Active rule.** The `%` prefix means "group." This grants all `wheel` members permission to run any command as any user on any host. **This is the line that matters.** |
| `# %wheel ALL=(ALL) NOPASSWD: ALL` | Commented out (disabled). If uncommented, `wheel` members could sudo **without entering a password**. Leave this commented unless you have a specific reason. |

#### Sudoers rule syntax decoded

```
%wheel  ALL=(ALL)  ALL
  │      │    │     │
  │      │    │     └── Which commands: ALL = no restrictions
  │      │    └──────── Run as which user: (ALL) = any user including root
  │      └───────────── From which host: ALL = any machine
  └──────────────────── Who: %wheel = members of the wheel group
```

> **Always use `visudo`** when editing `/etc/sudoers` directly. `visudo` locks the file and validates syntax before saving — a typo in `/etc/sudoers` can **lock every user out of sudo**. Never edit it with a plain text editor.

---

### Step 6 — Switch to `conadm` and test sudo

```bash
su - conadm
```

> The `-` flag launches a **login shell** for `conadm`, loading their full environment (`.bash_profile`, `$PATH`, etc.). Without `-`, you'd inherit the current user's environment — group changes may not be reflected.

Then run:

```bash
sudo whoami
```

**Expected output:**

```
[sudo] password for conadm:
root
```

#### Output explained

| Part | Meaning |
|---|---|
| `[sudo] password for conadm:` | sudo asks for `conadm`'s own password (not root's). This is correct behavior. |
| `root` | `whoami` printed the effective user after elevation. Sudo is working. ✅ |

> **If you get `conadm is not in the sudoers file`:** Log out completely and log back in as `conadm`. Group changes take effect at the **next login session** — they are not applied to currently active sessions.

---

### Step 7 — Exit back to original user

```bash
exit
```

> Returns you to `ec2-user`. Always exit cleanly — don't leave elevated sessions open.

---

## 🔄 Alternative Method: Direct `visudo` Entry

If you need to grant sudo to a user **without adding them to `wheel`** (e.g., restricted access), use:

```bash
sudo visudo
```

Add this line at the bottom of the file:

```
conadm  ALL=(ALL)  ALL
```

| Field | Meaning |
|---|---|
| `conadm` | The specific user this rule applies to |
| First `ALL` | From any host |
| `(ALL)` | Run as any user |
| Last `ALL` | Run any command |

> **Exam guidance:** Use `usermod -aG wheel` on the exam — it's faster, less error-prone, and the idiomatic RHEL approach. Reserve `visudo` for fine-grained per-user restrictions.

---

## ✅ Lab Checklist

- [ ] `sudo usermod -aG wheel conadm` ran without error
- [ ] `id conadm` shows `10(wheel)` in the groups field
- [ ] `grep wheel /etc/group` shows `conadm` as a member
- [ ] `sudo grep wheel /etc/sudoers` shows `%wheel ALL=(ALL) ALL` **uncommented**
- [ ] `sudo whoami` returns `root` when run as `conadm`
- [ ] Exited cleanly back to original user

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Using `-G` without `-a` | User stripped from all other groups silently | Always use `-aG` |
| Editing `/etc/sudoers` with `vi` directly | Syntax error locks out all sudo | Always use `visudo` |
| `wheel` line is commented out | sudo fails even with wheel membership | Remove `#` from `%wheel ALL=(ALL) ALL` in `visudo` |
| Group change not active | `conadm is not in the sudoers file` | Log out and back in as `conadm` to refresh session |
| `ls /home/conadm` without sudo | `Permission denied` | Use `sudo ls -la /home/conadm` |

---

## 📌 RHCSA Exam Strategy

- `usermod -aG wheel` is the **fastest, safest** method to grant sudo on RHEL — know it cold.
- **Never** edit `/etc/sudoers` directly — `visudo` only.
- The `%` prefix in sudoers means **group** — `%wheel` ≠ `wheel`.
- `NOPASSWD` variant exists but is a security risk — leave it commented unless explicitly required.
- Group changes require a **new login session** to take effect — always verify with `id` after `su -`.

---

## ➡️ Next Lab

**[Lab 22-1c: Verify sudo access](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm)**

---

## 🔗 Series Index

| Lab | Topic |
|---|---|
| ✅ [22-1a](https://github.com/kelvintechnical/Create-User-Account-Conadm) | Create user `conadm` |
| 👉 **22-1b** | Grant `conadm` full sudo rights ← *you are here* |
| [22-1c](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm) | Verify sudo access |
| [22-1d](https://github.com/kelvintechnical/Inspect-ubi9-with-skopeo) | Inspect ubi9 image with skopeo |
| [22-1e](https://github.com/kelvintechnical/Pull-ubi9-Image-with-podman) | Pull ubi9 image with podman |
| [22-1f](https://github.com/kelvintechnical/launch-container-interative-terminal) | Launch container with port mapping |
| [22-1g](https://github.com/kelvintechnical/run-commands-inside-terminal) | Run commands inside container |
| [22-1h](https://github.com/kelvintechnical/verify-port-mapping-from-host) | Verify port mapping from host |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
