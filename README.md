# Lab 22-1b: Grant Full Sudo Rights — `conadm`

**RHCSA EX200 Lab** | Series: Container Management (Lab 22)  
**Prerequisite:** Lab 22-1a complete (`conadm` user exists)  
**Time Estimate:** 5 minutes

---

## 🎯 Objective

Grant the `conadm` user full sudo privileges on server30.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| Edit sudoers file safely | `visudo` |
| Add user to wheel group | `usermod -aG wheel conadm` |
| Verify group membership | `id conadm` |
| Test sudo access as conadm | `sudo whoami` |

---

## 🔧 Steps

### Step 1 — Add `conadm` to the `wheel` group

On RHEL, members of the `wheel` group have full sudo rights by default.

```bash
sudo usermod -aG wheel conadm
```

> No output = success.

---

### Step 2 — Verify group membership

```bash
id conadm
```

**Expected output:**

```
uid=1002(conadm) gid=1003(conadm) groups=1003(conadm),10(wheel)
```

> `10(wheel)` must appear in the groups list. ✅

---

### Step 3 — Confirm wheel is enabled in sudoers

```bash
sudo grep wheel /etc/sudoers
```

**Expected output:**

```
%wheel  ALL=(ALL)  ALL
```

> The `%` prefix means "group". This line grants all wheel members full sudo. ✅

---

### Step 4 — Switch to `conadm` and test sudo

```bash
su - conadm
```

Then run:

```bash
sudo whoami
```

**Expected output:**

```
root
```

> If it returns `root`, sudo is working correctly. ✅

---

### Step 5 — Exit back to your original user

```bash
exit
```

---

## ✅ Lab Checklist

- [ ] `usermod -aG wheel conadm` ran without error
- [ ] `id conadm` shows `wheel` in groups
- [ ] `/etc/sudoers` has `%wheel ALL=(ALL) ALL` uncommented
- [ ] `sudo whoami` returns `root` as `conadm`
- [ ] Exited back to original user

---

## ⚠️ Common Pitfalls

| Mistake | Fix |
|---|---|
| Editing `/etc/sudoers` directly with `vi` | Always use `visudo` — syntax errors can lock you out |
| `wheel` line is commented out | Remove `#` from `%wheel ALL=(ALL) ALL` in `visudo` |
| Group change not active yet | Log out and back in as `conadm` to refresh group membership |
| `-aG` vs `-G` | Always use `-aG` — `-G` alone **replaces** all groups |

---

## 📌 Exam Tips

- RHEL enables `wheel` sudo by default — `usermod -aG wheel` is the fastest method.
- **Never** edit `/etc/sudoers` directly — always use `visudo`.
- If `sudo whoami` prompts for a password, that's correct behavior — `conadm` uses their own password.

---

## ➡️ Next Lab
**[Lab 22-1c: Verify sudo access](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm)**

---

## 🔗 Series Index
- ✅ [Lab 22-1a: Create conadm user](https://github.com/kelvintechnical/Create-User-Account-Conadm)
- 👉 **Lab 22-1b: Grant sudo rights** ← you are here
- [Lab 22-1c: Verify sudo access](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm)
- [Lab 22-1d: Inspect ubi9 with skopeo](https://github.com/kelvintechnical/Inspect-ubi9-with-skopeo)
- [Lab 22-1e: Pull ubi9 image](https://github.com/kelvintechnical/Pull-ubi9-Image-with-podman)
- [Lab 22-1f: Launch container with port mapping](https://github.com/kelvintechnical/launch-container-interative-terminal)
- [Lab 22-1g: Run commands inside container](https://github.com/kelvintechnical/run-commands-inside-terminal)
- [Lab 22-1h: Verify port mapping from host](https://github.com/kelvintechnical/verify-port-mapping-from-host)

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
