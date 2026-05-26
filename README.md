# Lab: Changing the Default Firewall Zone — Global Default + NIC Trust

**Series:** linux-ops-mastery — RHCSA Firewall
**Subjects covered:** Default zone semantics, `firewall-cmd --get-default-zone`, `--set-default-zone`, moving an interface from `public` to `dmz` or `internal` with `--change-interface`, verifying with `--get-active-zones`, runtime vs permanent preview (`--permanent` flags named but full persistence pattern in Lab 61)
**Career arcs covered:** RHCSA (EX200 loves “set default zone to …”), RHCE (playbook idempotency around defaults), SRE (hardening baselines), DevOps (immutable infra still boots with a default), AI/MLOps (segmenting admin from data NIC defaults)
**Prerequisite:** Lab **firewalld-zones** (vocabulary) or equivalent comfort listing zones
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 snapshot active state · 2–3 move NIC trust · 4 change global default · 5 verify services inherited · 6 capstone + full revert

---

## Objective

Two different knobs confuse beginners:

1. **Default zone** — the fallback label for traffic that is not matched elsewhere.
2. **Interface binding** — which zone actually owns a specific NIC **right now**.

This lab practices both because exam scenarios often say: “Put `ens160` in `internal` **and** make `internal` the default.” You will move one live interface out of `public` into `dmz` **or** `internal`, then change the global default, verify inherited `services:` lines, and **fully revert** so your VM returns to lab-safe `public`.

> **Lab safety note:** Changing defaults on a remote SSH session can lock you out if the new zone omits `ssh`. **Stay on console** or ensure `ssh` remains listed under your target zone **before** applying changes. Task 6 always reverts.

---

## Concept: Default Zone ≠ “The Only Zone”

```
Traffic arrives on ens160
        │
        ▼
  Does a source rule map it? ──yes──► that zone
        │ no
        ▼
  Is ens160 bound to a zone? ──yes──► that zone (e.g., dmz)
        │ no
        ▼
  Fall back to DEFAULT ZONE (e.g., internal)
```

Changing **default** without binding interfaces still alters fallback behavior for unmapped traffic — powerful and risky.

> **Why this matters:** A “small” default change can accidentally expose services that were only allowed in `public`’s tight bundle — or remove `ssh` from the path you rely on. Read `services:` **before** committing.

---

## 📜 Why Default Zones Exist — The Story

Early Linux firewalls asked operators to think purely in chains and rules — powerful, but error-prone under pressure. `firewalld` introduced **zones** so policy could follow the way humans already described networks: “this cable is the corporate LAN” vs “this one faces the internet.”

The **default zone** is the answer to: “If I have not classified this traffic yet, how paranoid should I be?” Vendors ship conservative answers — historically **`public`** on Red Hat–family servers — so first boot is safer than “open by default.”

Enterprise rollouts often **tighten or loosen** defaults to match rack location: a hypervisor management NIC might default to `internal`, while guest-facing bridges stay `public` or `dmz`. Red Hat has shipped `firewalld` as the supported firewall service since **RHEL 7**, evolving the backend while keeping the zone abstraction stable for automation and human operators alike.

> **The point of the story:** Defaults encode organizational trust. Changing them is a security decision, not a syntax exercise.

---

## 👪 The Default Zone Family — Who Lives There

### By command

| Goal | Runtime command |
|---|---|
| Read default | `firewall-cmd --get-default-zone` |
| Set default | `firewall-cmd --set-default-zone=ZONE` |
| Read active bindings | `firewall-cmd --get-active-zones` |
| Move NIC between zones | `firewall-cmd --zone=TARGET --change-interface=IFACE` |

### By “what moves?”

| Object | Effect |
|---|---|
| Interface binding | That NIC inherits TARGET zone’s `services`/`ports` |
| Default zone only | Unmapped traffic inherits new fallback |

### By safety checks

| Check | Command fragment |
|---|---|
| SSH still allowed? | `firewall-cmd --info-zone=TARGET \| grep services` |
| Active view | `firewall-cmd --list-all --zone=TARGET` |

> **The point of the family tree:** Bindings answer “**which NIC**”; default answers “**everything else**.”

---

## 🔬 The Anatomy of `--get-active-zones` — In One Diagram

```
$ firewall-cmd --get-active-zones
public
  interfaces: ens160
dmz
  interfaces: ens192
```

Read down as **pairs**: each stanza is a **zone name**, indented lines list **bindings** currently enforcing that zone’s bundle.

```
public          ← zone label
  interfaces:   ← binding type
  ens160         ← concrete NIC
```

> **Reading rule:** After you `--change-interface`, the NIC disappears from the old stanza and appears under the new zone header.

---

## 📚 Default Zone & Interface Move Reference Table

| Task | Command | Notes |
|---|---|---|
| Show NICs | `ip -br link` | Pick the interface you SSH through carefully |
| Move NIC | `firewall-cmd --zone=dmz --change-interface=ens160` | Runtime move |
| Set default | `firewall-cmd --set-default-zone=internal` | Global fallback |
| Verify | `firewall-cmd --get-active-zones` | Confirms bindings |
| Deep verify | `firewall-cmd --list-all --zone=dmz` | Shows inherited services |
| Revert default | `firewall-cmd --set-default-zone=public` | Common safe baseline |

> **Rule one of moves:** Ensure `ssh` (or your access path) appears in the **target** zone’s `services:` before moving your only admin NIC.

---

## 🧪 Extended Verification Playbook (Optional Depth)

| Situation | Command | What “good” looks like |
|---|---|---|
| Prove `firewalld` owns the NIC | `nmcli -f GENERAL.ZONE dev show "$IFACE"` | May echo `firewalld` integration; compare to `get-active-zones` |
| See XML backing | `ls -1 /etc/firewalld/zones/` | Zone files appear after permanent edits (other labs) |
| Confirm D-Bus alive | `firewall-cmd --panic-off` | Should return `success` (also proves you are not panicked) |
| Packet path thought experiment | `ping -c1 127.0.0.1` | Local traffic — unrelated to NIC zones but sanity-checks network stack |
| Secondary sanity | `ss -lntp \| head` | Shows listeners; pair with zone `services` to reason about exposure |
| Log signals | `journalctl -u firewalld -n 20 --no-pager` | Recent daemon errors after failed moves |
| Version stamp | `rpm -q firewalld` | Baseline package level for classroom drift |
| Rollback rehearsal | `firewall-cmd --get-active-zones` before/after each move | Identical pattern to change-window evidence |

Use this table when you want **extra interview depth** beyond the six core tasks. None of these rows are required for a minimal exam pass — they train professional thoroughness.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Tasks combine EX200-style “change default” + “attach interface to zone.” |
| **RHCE candidate** | Ansible must model both `default_zone` and `interface` parameters without drift. |
| **SRE / Platform** | Defaults are how golden images silently diverge — detect with startup scripts. |
| **DevOps** | Terraform/Ansible order matters: open service → move NIC → change default. |
| **AI / MLOps** | Management vs data plane NICs often need different defaults — practice here first. |

---

## 🔧 The 6 Tasks

> Mutates **runtime** firewall — Task 6 returns host to `public` safety.

---

### Task 1 — Capture baseline: default zone, active zones, and NIC names

**Purpose:** Record “before” so revert is painless and provable.

```bash
sudo -i
ip -br link
echo "DEFAULT=$(firewall-cmd --get-default-zone)"
firewall-cmd --get-active-zones
IFACE=$(ip -br link | awk '!/lo/ && $2=="UP" {print $1; exit}' | sed 's/@.*//')
echo "PRIMARY_IFACE=$IFACE"
```

**Human-Readable Breakdown:** Choose the first non-loopback **UP** interface as `$IFACE` for the lab. If you only have `lo`, bring a NIC up in hypervisor settings.

**Reading it left to right:** `ip -br link` prints `IFNAME STATE ADDR`. Awk skips `lo`, requires `UP`, prints first match.

**The story:** Every exam VM has different `ens*` names — never hardcode `eth0` mentally.

**Expected output:**

```text
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens160           UP             00:0c:29:aa:bb:cc <BROADCAST,MULTICAST,UP,LOWER_UP>
DEFAULT=public
public
  interfaces: ens160
PRIMARY_IFACE=ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `ip -br link` | Brief link states |
| `--get-default-zone` | Current global default |
| `--get-active-zones` | Zone → bindings map |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `PRIMARY_IFACE` empty | Bring an interface `UP`; verify with `nmcli dev status` |
| No `firewalld` | `systemctl start firewalld` |

---

### Task 2 — Core A: prove `ssh` exists in target zone before moving anything

**Purpose:** Avoid lockout — always read target services first.

```bash
TARGET=internal
firewall-cmd --info-zone="$TARGET" | grep -E 'services:|ports:|target:'
```

**Human-Readable Breakdown:** Swap `TARGET` to `dmz` if your instructor prefers — both ship service bundles; **verify** `ssh` is present for remote labs.

**Reading it left to right:** `grep` isolates the three lines that decide connectivity fate.

**The story:** This thirty-second habit has prevented more outages than any vendor feature.

**Expected output:**

```text
  target: default
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports:
```

**Switches**

| Token | Meaning |
|---|---|
| `TARGET=internal` | Shell variable reused later |
| `--info-zone` | Rich stanza for one zone |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No `ssh` in target | Add service first (Lab 58) or pick another zone / use console |

---

### Task 3 — Core B: move `$IFACE` from `public` to `$TARGET` at runtime

**Purpose:** Execute `--change-interface` and observe active zone map shift.

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone="$TARGET" --change-interface="$IFACE"
firewall-cmd --get-active-zones
firewall-cmd --list-all --zone="$TARGET" | head -n 20
```

**Human-Readable Breakdown:** First snapshot proves old binding; post-change output should list `$IFACE` under `$TARGET`.

**Reading it left to right:** `--change-interface` **moves** binding; it is not a second parallel label.

**The story:** This is the fastest “make this NIC LAN-trusted” motion on a live host — still runtime-only until Lab 61’s permanent pattern.

**Expected output:**

```text
internal
  interfaces: ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `--zone=Z --change-interface=IF` | Move IF into zone Z |
| `--list-all --zone` | Confirms inherited policy view |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ZONE_ALREADY_SET` style message | Idempotent success — verify with `get-active-zones` |
| `NO_INTERFACE` | `$IFACE` wrong — re-derive Task 1 |

---

### Task 4 — Change the global default zone and observe fallback semantics

**Purpose:** Practice `--set-default-zone` **after** SSH safety is confirmed.

```bash
firewall-cmd --set-default-zone="$TARGET"
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
```

**Human-Readable Breakdown:** Global default now matches the lab narrative (`internal` or `dmz` path). Active map still shows explicit bindings.

**Reading it left to right:** `--set-default-zone` returns success quickly — always follow with `get-default-zone`.

**The story:** Examiners love pairing **binding** + **default** in one story paragraph — you rehearse both muscles.

**Expected output:**

```text
success
internal
internal
  interfaces: ens160
```

**Switches**

| Token | Meaning |
|---|---|
| `--set-default-zone` | Changes runtime default |
| `--get-default-zone` | Read-back verification |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Lost SSH | Physical/console access → move NIC back to `public` or enable ssh service in zone |

---

### Task 5 — Edge case: compare services on `public` vs `$TARGET` for drift

**Purpose:** Build the security reviewer reflex — what opened/closed when the zone changed?

```bash
echo "-- public --"; firewall-cmd --list-services --zone=public
echo "-- target --"; firewall-cmd --list-services --zone="$TARGET"
```

**Human-Readable Breakdown:** `list-services` is compact — ideal for diffing mental models.

**Reading it left to right:** Each line is a **named service** opening (maps to ports via `/usr/lib/firewalld/services`).

**The story:** “Why did backups break?” often equals “`samba-client` disappeared when we left `internal`.”

**Expected output:**

```text
-- public --
cockpit dhcpv6-client ssh
-- target --
cockpit dhcpv6-client mdns samba-client ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--list-services --zone=` | Prints space-separated service names |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Identical lists | Rare — still valid; note `ports:` differences with `--list-ports` |

---

### Task 6 — Capstone narrative + full revert to `public`

**Purpose:** Verbalize the exam answer path, then **restore baseline**: default `public`, interface back in `public`.

```bash
# Capstone verification snapshot
{
  echo "DEFAULT=$(firewall-cmd --get-default-zone)"
  firewall-cmd --get-active-zones
  firewall-cmd --list-all --zone="$TARGET" | head -n 15
} | tee /tmp/default-zone-lab.txt
cat /tmp/default-zone-lab.txt
```

**Human-Readable Breakdown:** Snapshot is your proctor voice — “this interface is internal-trusted; default is internal.”

**Reading it left to right:** Group command bundle → `tee` file → `cat` proof.

**The story:** RHCSA wants the **end state**, not poetry — still, practice narrating while typing.

**Expected output:**

```text
DEFAULT=internal
internal
  interfaces: ens160
...
```

**Switches**

| Token | Meaning |
|---|---|
| `tee /tmp/...` | Temporary evidence file |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `tee` permission | Run as root (`sudo -i`) |

**Cleanup**

```bash
firewall-cmd --set-default-zone=public
firewall-cmd --zone=public --change-interface="$IFACE"
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
rm -f /tmp/default-zone-lab.txt
```

---

## 🔍 Default Zone Change Decision Guide

```
Need to retarget trust?
  │
  ├── "Only one NIC should relax"
  │       └── --zone=TARGET --change-interface=IFACE
  │
  ├── "Global fallback should relax/tighten"
  │       └── --set-default-zone=ZONE
  │
  ├── "Am I about to lose SSH?"
  │       └── grep ssh in --info-zone=TARGET BEFORE moving NIC
  │
  ├── "Make it survive reboot"
  │       └── Lab 61 + --permanent/--reload pattern
  │
  └── "Audit what changed"
          └── diff --list-services public vs TARGET
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Record default zone, active zones, derive `$IFACE`
- [ ] 02 Confirm `ssh` exists in `$TARGET` zone info
- [ ] 03 Move `$IFACE` to `$TARGET` with `--change-interface`
- [ ] 04 Set default zone to `$TARGET` and verify triple-read
- [ ] 05 Compare `public` vs `$TARGET` service lists
- [ ] 06 Snapshot story file, then Cleanup revert to `public` + delete `/tmp` file

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Move NIC before reading services | SSH hang | Console recover + move back |
| Confuse default vs binding | “Changed default, NIC still shows old zone?” | Read `--get-active-zones` carefully |
| Hardcode `eth0` | Command no-op / wrong NIC | Use `ip -br link` |
| Skip Task 6 revert | Next lab inherits weird state | Always run Cleanup block |
| Assume runtime == boot | Surprise after reboot | Follow permanent labs |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize pair: `--change-interface` + `--set-default-zone` + verification trio (`get-active-zones`, `get-default-zone`, `list-all --zone`).

**RHCE candidate**
- Discuss idempotency: playbook should declare desired zone per interface, not blindly toggle defaults.

**SRE / Platform interview**
- Explain lockout prevention and how you’d automate service presence checks pre-change.

**DevOps**
- Encode interface names via facts (`ansible_default_ipv4.interface`) not literals.

**AI / MLOps**
- Treat default zone changes like routing table edits — announce, verify, rollback ready.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [firewalld-zones](https://github.com/kelvintechnical/firewalld-zones) | Zone vocabulary |
| [reassign-interfaces-zones](https://github.com/kelvintechnical/reassign-interfaces-zones) | Deeper permanent interface moves |
| [firewalld-add-services](https://github.com/kelvintechnical/firewalld-add-services) | When default bundles lack a daemon |
| [active-firewall-zones](https://github.com/kelvintechnical/active-firewall-zones) | Ops audit of bindings |

---

## 🎓 After the Lab — 60-Second Oral Exam

Answer out loud without scrolling:

- What is the difference between **default zone** and **interface binding**?
- Why must you read `services:` before moving your SSH NIC?
- Which command proves an interface moved: `--list-all` or `--get-active-zones`?
- What two commands return you to `public` in the Cleanup block?
- What log unit tells you if `firewalld` rejected a D-Bus change?

If any answer wobbles, redo Tasks 1–3 slowly — speed without correctness is how points evaporate.

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
