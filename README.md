# 🛰️ GreenOptic: How We Absolutely Deleted an ISP (Emotionally)

*aka: “Totally Professional Incident Response 🙃”*

---

## ⚡ Hook: What if your ISP’s security was held together with duct tape?

You know that one ISP that keeps sending you “we take security very seriously” emails?  
GreenOptic is the **spider‑verse variant** of that ISP where they say it… and **don’t mean it at all**. 😈

This is the story of how we:

- Turned a **customer portal** into a free file browser 🗂️
- Used **DNS** like a gossip channel to learn their secrets 📡
- Read their **recovery plan**, which absolutely needed its own recovery plan 📉
- Stole creds from **pcap files** because why not 🤦
- Promoted ourselves from `alex` to **root**, using **Wireshark group membership** as a feature
- Then wrote a **Windows script** that pretends to be Mr. Robot while just SSHing in as root in the background 🎭

If compliance asks: this was a “resilience exercise”. If HR asks: this was “a misunderstanding”.

---

## 🧑‍💻 Cast of Characters

- **You** – Overcaffeinated hacker, main character energy ☕
- **GreenOptic** – ISP with the security posture of a cardboard door 🚪
- **`sam`, `alex`, `staff`** – NPCs who unknowingly speed‑ran credential sharing
- **`root`** – Final boss, but password hygiene is on easy mode

---

## 1️⃣ Recon – “What ports are you hiding, my beloved?”

We started with the classic: **scan everything and judge them silently.**

We hit `192.168.101.10` with a full `nmap` scan and discovered:

- **21/tcp** – FTP (`vsftpd 3.0.2`) 📁
- **22/tcp** – SSH (`OpenSSH 7.4`) 🔐
- **53/tcp** – DNS (`BIND 9.11.4`) 🧠
- **80/tcp** – Apache + PHP (`GreenOptic` website) 🌐
- **10000/tcp** – Webmin (`MiniServ`) 🛠️

So right off the bat:

> “Hi, we’re GreenOptic and we’ve exposed **literally every classic attack surface** in one convenient appliance.”

Challenge accepted. 😏

---

## 2️⃣ Marketing Site – “Welcome to GreenOptic, please hack responsibly”

We browsed to `http://192.168.101.10/` and got:

- A gorgeous brochure site.
- Buttons to `statement.html` (about a “cyber attack” 🥲) and an **account portal**.

We unleashed directory brute forcing and found:

- `/account/` – our future best friend
- Static stuff: `/css`, `/img`, `/js`, `/vendor`, etc.

Going to `/account/` dropped us on a login form with this suspicious URL:

```text
/account/index.php?include=cookiewarning
```

An `include` parameter in PHP.  
With user‑controlled value.  
In 2020‑something.  
> Oh no baby, what is you doing. 😬

---

## 3️⃣ LFI – “Can I have your `/etc/passwd`, with extra shame?”

We gently tested for **Local File Inclusion**:

```text
/account/index.php?include=../../../../etc/passwd
```

The page responded with the normal HTML… and then **dumped `/etc/passwd` into the response** like it was a fun fact.

We now had:

- System users like `sam`, `alex`, `terry`, `monitor` 🍽️
- Proof that `include` was basically: `include($_GET['include'])` with zero adult supervision.

> Customer portal: designed for customers.  
> Also customer portal: silently serving local system files to anyone with a URL bar.

---

## 4️⃣ DNS – “Tell me your secrets, BIND”

Port 53 was open with `BIND`, and the site mentioned GreenOptic’s domain. So we tried a **zone transfer**, because sometimes the universe rewards optimism.

From our attack box:

```text
dig @192.168.101.10 greenoptic.vm axfr
```

And BIND said: “Sure, have everything. I trust you.” 🫡

We saw entries like:

- `websrv01.greenoptic.vm`
- **`recoveryplan.greenoptic.vm`** 🤔

We added them to `/etc/hosts`:

```text
192.168.101.10 greenoptic.vm websrv01.greenoptic.vm recoveryplan.greenoptic.vm
```

Because any domain named **`recoveryplan`** is either super secure… or the exact opposite. Spoiler: it’s the opposite.

---

## 5️⃣ Recovery Plan – “Step 1: Get Hacked”

We browsed to:

```text
http://recoveryplan.greenoptic.vm/
```

And got hit with **HTTP Basic Auth**.  
Nothing too fancy, probably backed by `.htpasswd`.

We thought: *“If only we had a way to read arbitrary files on this host…”*  
Then remembered: **we literally do.** Thanks, LFI. 🥰

Using the LFI:

- We used Apache configs (also via LFI) to track down the path to `.htpasswd`.
- Then grabbed the file with:

  ```text
  /account/index.php?include=../../../../path/to/.htpasswd
  ```

The contents:

```text
staff:$apr1$YQNFpPkc$rhUZOxRE55Nkl4EDn.1Po.
```

GreenOptic had:

- A user: `staff`
- A hash: Apache MD5 (`$apr1$`) aka *“Please feed me to John the Ripper.”*

We obliged. 🧑‍🍳

---

## 6️⃣ Hash Cracking – “John, do your thing”

On our attack box:

```text
echo 'staff:$apr1$YQNFpPkc$rhUZOxRE55Nkl4EDn.1Po.' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

John spat out:

- `staff : wheeler`

So the **recovery plan** was protected by **“wheeler”**.  
Is that a password, a skateboard trick, or both? 🛹

We logged into `recoveryplan.greenoptic.vm` as `staff:wheeler` and moved in.

---

## 7️⃣ phpBB – “Security Forum, Zero Security”

Inside `recoveryplan` we found a **phpBB** forum. Understandable, admins love forums.

We discovered:

- A thread from an admin about their “incident response”.
- An attached archive: **`dpi.zip`**.
- A note that the zip is password protected, and the password was emailed to **Sam**.

So:

> Step 1: Have a breach.  
> Step 2: Make a forum about it.  
> Step 3: Attach sensitive files.  
> Step 4: Email passwords in plaintext.  
> Step 5: Wonder how this keeps happening.

We downloaded `dpi.zip` and went hunting for Sam’s inbox.

---

## 8️⃣ Reading Sam’s Email – “Dear Sam, Thanks for the Password 💌”

Once again, the LFI took the wheel. We tried common mail paths:

```text
/account/index.php?include=../../../../var/mail/sam
/account/index.php?include=../../../../var/spool/mail/sam
```

Eventually, we landed on Sam’s mailbox… and there it was:

- An email containing the password for `dpi.zip`.

We copied the password, gently judged Sam’s OPSEC, and unzipped `dpi.zip` on our attack box to reveal:

- **`dpi.pcap`** – a packet capture, full of network sadness.

---

## 9️⃣ DPI.pcap – “Your Network Traffic, Our Credential Store”

We opened `dpi.pcap` in Wireshark and filtered for FTP:

- Filter: `ftp` or `tcp.port == 21`

We saw:

- `USER alex`
- `PASS <ftp_password>`

And thus, an FTP user was born.

Security level: sending creds over FTP and then archiving them in a zip whose password is emailed to someone whose mail is readable via LFI. 🔥

---

## 🔟 FTP & SSH – “Hi Alex, We Live Here Now”

Using the pcap creds:

```text
ftp 192.168.101.10
# USER alex
# PASS <password from pcap>
```

Inside Alex’s FTP directory we found:

- A `user.txt` flag 🎉
- A helpful note basically saying: “Try these creds on SSH too.”

So, obviously:

```text
ssh alex@192.168.101.10
# password: same as FTP
```

And just like that, we went from anonymous internet goblin to **Alex, local user**.

> Dear Alex, thank you for your service. You did nothing wrong. Except the FTP thing.

---

## 1️⃣1️⃣ Priv Esc Setup – “Why Can Alex Use Wireshark? 👀”

On the box as `alex`:

```text
id
```

We saw:

- `uid=1002(alex)`
- `groups=1002(alex),994(wireshark)`

Alex was in the **`wireshark` group**.

This means Alex could capture packets on the system. And this system just so happens to send **SMTP authentication** (for root) on a loop.

> You know that meme: “Modern problems require modern solutions”?  
> Here it was more like: “Modern monitoring requires complete compromise.”

---

## 1️⃣2️⃣ Sniffing Root’s Password – “SMTP, but make it base64”

The intended path on this box:

1. Use Wireshark (or CLI tools) as `alex` to capture SMTP traffic (`tcp port 25`).
2. Wait for a recurring **SMTP AUTH** from root.
3. Extract the **base64** blob from the AUTH command.
4. Decode to get username/password.

Example captured token:

```text
AHJvb3QAQVNmb2pvajJlb3p4Y3p6bWVkbG1lZEFTQVNES29qM28=
```

Decode it:

```text
echo -n 'AHJvb3QAQVNmb2pvajJlb3p4Y3p6bWVkbG1lZEFTQVNES29qM28=' | base64 -d
```

Which yields something like:

```text
root\0ASfojoj2eozxczzmedlmedASASDKoj3o
```

Extracting the important part:

- **Root password**: `ASfojoj2eozxczzmedlmedASASDKoj3o`

Root is out here broadcasting its password in base64 over SMTP like a Twitch stream.

---

## 1️⃣3️⃣ Root Login – “No Sudo, Just Vibes”

Armed with the stolen password, we did the obvious:

```text
ssh root@192.168.101.10
# password: ASfojoj2eozxczzmedlmedASASDKoj3o
```

Then checked:

```text
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# Congratulations on getting root! (plus ASCII art and a love letter from the author)
```

We went from:

- Anonymous rando ➜ LFI gremlin ➜ DNS goblin ➜ Recovery plan tourist ➜ Sam’s mailman ➜ Alex ➜ **root**.

Skill issue? Not ours. 😎

---

## 1️⃣4️⃣ Windows Flex – “Auto‑Root Orchestrator 🎭”

To add some cinematic flair, we wrote a **Windows Python script** that:

- Shows a big colorful banner: *“GreenOptic Auto-Root Orchestrator”* 🌈
- Displays fake “elite hacker” stages with spinners:
  - “Scanning GreenOptic perimeter”
  - “Abusing LFI in customer portal”
  - “Forging privileged SSH session”
- In reality, it simply:
  - Uses `paramiko` to SSH into `192.168.101.10` as **root** with the password we already stole.
  - Prints `id` and `/root/root.txt`.
  - Drops you into an interactive `root@greenoptic#` prompt where everything you type runs on the victim.

So, from your Windows host, you can go:

```text
cd "how i solved the lab"
python greenoptic_root_windows.py
```

And watch a **Hollywood hacking montage** while a regular SSH session does the actual work.

Your screen: 🔥💻 “We’re breaching the mainframe!”  
Reality: `paramiko` going “hello yes root login please”.

---

## 🧠 Lessons Learned (For GreenOptic, Not You)

What GreenOptic accidentally taught us:

- **Do not**:
  - Use LFI‑prone `include` parameters in prod.
  - Allow DNS AXFR to whoever asks nicely.
  - Protect your recovery system with `.htpasswd` and a RockYou‑grade password.
  - Store credentials in pcaps, zip them, and email the zip password.
  - Put a normal user in the `wireshark` group on a host sending root SMTP auth.

- **Do**:
  - Assume attackers are lazy **but** persistent.
  - Assume they will turn your mistakes into content.

We chained:

> Recon ➜ LFI ➜ DNS AXFR ➜ `.htpasswd` ➜ phpBB + `dpi.zip` ➜ Sam’s mail ➜ FTP creds ➜ SSH as alex ➜ Wireshark group ➜ sniff SMTP ➜ root password ➜ SSH as root.

With a final sprinkle of:

- **Windows auto‑root script**
- **Colored terminal nonsense**
- **Way too much fun for a “serious” assessment**

GreenOptic: **0**  
You: **root + a comedy writeup** 💥
