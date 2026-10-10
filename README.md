# Migrating a Legacy Web App to AWS

**A hands-on rehost ("lift-and-shift") migration of a WordPress site onto Amazon EC2, ending with a tested backup and restore of the database.**

Built as Project 1 of the Cloud Engineering program at **1% University** · October 2026

![Architecture](architecture.svg)

| | |
|---|---|
| **Goal** | Move a company's legacy website to AWS so it is easier to manage, more secure, and ready to scale |
| **Strategy** | Rehost (lift-and-shift), one of the "7 Rs" of migration |
| **Stack** | Amazon EC2 (t3.micro, Ubuntu), Apache2, PHP, MySQL, WordPress |
| **Outcome** | Live WordPress site on AWS, database backed up, restored, verified 12 tables for 12, and saved off the server before decommissioning |
| **Skills shown** | EC2, security groups, SSH key management, Linux administration, LAMP stack, MySQL, mysqldump and restore, troubleshooting, documentation |

---

## Contents

1. [The scenario](#1-the-scenario)
2. [Chapter 1: Building the landing spot](#chapter-1-building-the-landing-spot)
3. [Chapter 2: Getting in the door](#chapter-2-getting-in-the-door)
4. [Chapter 3: Installing the engine](#chapter-3-installing-the-engine)
5. [Chapter 4: Preparing the database](#chapter-4-preparing-the-database)
6. [Chapter 5: Moving the application](#chapter-5-moving-the-application)
7. [Chapter 6: The config file that wouldn't save](#chapter-6-the-config-file-that-wouldnt-save)
8. [Chapter 7: Go-live](#chapter-7-go-live)
9. [Chapter 8: Protecting the data before shutdown](#chapter-8-protecting-the-data-before-shutdown)
10. [Troubleshooting log](#troubleshooting-log)
11. [What I'd do next in production](#what-id-do-next-in-production)
12. [Repository structure](#repository-structure)

---

## 1. The scenario

A company runs its website on an aging, on-premises server. It is hard to patch, can't grow with traffic, and depends on hardware nobody wants to maintain. My job was to move the site to AWS with a clear, repeatable process, prove it works, and protect the data before the old environment is retired.

I chose a **rehost** strategy: move the application as it is onto a cloud server first, then modernize later. It is the fastest, lowest-risk first step in most real migrations.

---

## Chapter 1: Building the landing spot

The site needed a home in the cloud. I launched an **EC2 instance** (t3.micro, Ubuntu) in **us-east-1** and gave it a **security group** acting as a firewall:

- **SSH (port 22)** allowed **only from my own IP address**, so nobody else can even attempt to log in
- **HTTP (port 80)** open to everyone, so visitors can reach the site

![EC2 instance running](01-ec2-instance-running.png)
![EC2 instance summary](02-ec2-instance-summary.png)
![Security group inbound rules](03-security-group-inbound-rules.png)

> **Why it matters:** least-privilege network access is the first line of defense. Locking SSH to a single IP removes the most common attack surface on a public server.

---

## Chapter 2: Getting in the door

I connected from Windows PowerShell using an SSH key pair. Windows OpenSSH refused the key at first: *"UNPROTECTED PRIVATE KEY FILE."* Other accounts could read it, and SSH won't use a key that isn't private.

I fixed it by stripping inherited permissions and granting read access to my account only:

```powershell
icacls .\wp-key-bm.pem /inheritance:r
icacls .\wp-key-bm.pem /grant:r "$(whoami):(R)"
ssh -i "wp-key-bm.pem" ubuntu@<public-ip>
```

![Fixing key permissions](04-ssh-key-permission-fix.png)
![Connected over SSH](05-ssh-connected.png)

---

## Chapter 3: Installing the engine

With access established, I updated the OS and installed the **LAMP** stack: **L**inux, **A**pache, **M**ySQL, **P**HP.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 -y
sudo apt install php libapache2-mod-php php-mysql -y
sudo apt install mysql-server -y
sudo systemctl enable --now apache2 mysql
sudo mysql_secure_installation
```

Both services came up healthy, and Apache's default page loading at the public IP proved the server **and** the security group were working. I then hardened MySQL: I removed anonymous users, disabled remote root login, and dropped the test database.

![Apache and MySQL running](06-apache-mysql-running.png)
![Apache default page](07-apache-default-page.png)
![MySQL secured](08-mysql-secure-installation.png)

---

## Chapter 4: Preparing the database

WordPress needs its own database and a **dedicated, least-privilege user**. It should never run as root.

```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY '<strong-password>';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
FLUSH PRIVILEGES;
```

My first `CREATE USER` was rejected with **error 1819**: the placeholder password failed MySQL's password policy, which I had enabled myself during hardening. The user was never created, so the follow-up `ALTER USER` failed too. Re-running `CREATE USER` with a compliant password fixed it.

![MySQL error 1819](09-mysql-error-1819.png)
![User created and granted](10-mysql-user-created.png)

> **Lesson:** a security control doing its job can look like a bug. Reading the error code told me exactly why.

---

## Chapter 5: Moving the application

I downloaded WordPress, moved it into Apache's web root, and gave the web server ownership of the files.

```bash
wget https://wordpress.org/latest.tar.gz
tar -xvzf latest.tar.gz
sudo mv wordpress/* /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

Before and after: the web root held only Apache's `index.html`; afterward it held the full WordPress application.

![Web root before](11-webroot-before-wordpress.png)
![WordPress files in place](12-wordpress-files-in-place.png)

---

## Chapter 6: The config file that wouldn't save

WordPress connects to MySQL through `wp-config.php`. I opened it in **nano** to set the database name, user, and password, but nano's save prompt wouldn't accept the Enter key in my Windows terminal.

![wp-config in nano](13-wp-config-in-nano.png)
![Save prompt stuck](15-nano-save-stuck.png)
![Placeholders still present](17-grep-placeholders.png)

Instead of fighting the editor, I verified the file with `grep`, then made the edit **non-interactively with `sed`**, which is also how you'd do it in automation:

```bash
sudo sed -i 's/database_name_here/wordpress_db/; s/username_here/wordpress_user/; s/password_here/<strong-password>/' /var/www/html/wp-config.php
grep DB_ /var/www/html/wp-config.php
sudo rm /var/www/html/index.html
sudo systemctl restart apache2
```

![sed fix confirmed](18-sed-fix-and-restart.png)

---

## Chapter 7: Go-live

Browsing to the public IP brought up the WordPress installer. That alone proved WordPress could reach MySQL, since a bad connection shows an error instead. I finished setup and logged in to the dashboard. The site was live on AWS.

![WordPress installer](19-wordpress-installer.png)
![WordPress dashboard](21-wordpress-dashboard.png)
![WordPress settings](24-wordpress-general-settings.png)
![Live site](25-live-site.png)

---

## Chapter 8: Protecting the data before shutdown

A migration isn't finished until the data is **backed up, restored, and verified**. Before terminating the instance, I ran a full dump-and-load test.

![Data dump and load flow](data-dump-and-load.svg)

### The twist: MySQL kept dying

My first `mysqldump` failed with **error 2002**: MySQL wasn't running. `systemctl status` showed why: **`Failed with result 'oom-kill'`**. With only 1 GB of RAM, the t3.micro had run out of memory, and Linux killed MySQL **13 times**.

![Socket error](28-mysqldump-socket-error.png)
![OOM kill](29-mysql-oom-kill.png)

I added a 1 GB swap file to give MySQL breathing room, confirmed it with `free -h`, and re-ran the dump.

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h
```

### Dump

```bash
sudo mysqldump --single-transaction --routines --triggers wordpress_db > ~/wordpress_db_backup.sql
ls -lh ~/wordpress_db_backup.sql          # 176K
grep -c "CREATE TABLE" ~/wordpress_db_backup.sql   # 12
```

`--single-transaction` takes a consistent snapshot without locking tables, so the site stays usable during the backup.

![Swap and dump](30-swap-and-dump.png)

### Load and verify

```bash
sudo mysql -e "CREATE DATABASE wordpress_db_restore;"
sudo mysql wordpress_db_restore < ~/wordpress_db_backup.sql
sudo mysql -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='wordpress_db';"
sudo mysql -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='wordpress_db_restore';"
```

![Restore verified](31-restore-verified.png)

### Save it off the server

```powershell
scp -i "wp-key-bm.pem" ubuntu@<public-ip>:~/wordpress_db_backup.sql "$HOME\OneDrive\Desktop\Project 1"
```

![Backup downloaded](32-backup-downloaded.png)

### Results

| Check | Result |
|---|---|
| Backup file on server | 176K |
| Tables in dump | 12 |
| Tables in original database | 12 |
| Tables in restored database | 12, a match |
| Transfer to my PC | 100%, 179,267 bytes |

With a verified backup safely off the instance, the server could be terminated without risk of data loss.

---

## Troubleshooting log

| Problem | Root cause | Fix |
|---|---|---|
| SSH: `UNPROTECTED PRIVATE KEY FILE` | Key file readable by other Windows accounts | `icacls` to remove inheritance and any extra entries, read access for my user only |
| SSH: identity file not accessible | Placeholder path typed literally | Located the key with `Get-ChildItem -Recurse` and used the full path |
| PowerShell `Get-Process` error | Pasted the `PS C:\>` prompt along with the command | Typed only the command |
| MySQL `ERROR 1819` | Placeholder password failed the validation policy | Re-ran `CREATE USER` with a compliant password |
| nano wouldn't save | Enter key not registering at nano's prompt in Windows Terminal | Edited `wp-config.php` with `sed`, verified with `grep` |
| `mysqldump` error 2002 | MySQL OOM-killed on a 1 GB instance | Added 1 GB swap, restarted MySQL, re-ran the dump |
| "Sudo is disabled on this machine" | Linux command run in a Windows window | Reconnected over SSH; Linux commands run only at the `ubuntu@...$` prompt |
| Shell stuck at `>` | Trailing `\"` left a quote open | `Ctrl+C`, ran `scp` from PowerShell without the trailing backslash |

---

## What I'd do next in production

This project is a deliberate first step. To move from **rehost** toward **replatform**, I would:

- **Move MySQL to Amazon RDS** for managed backups, patching, and Multi-AZ failover, loading this same dump or using **AWS DMS** for near-zero-downtime replication
- **Add an Application Load Balancer and Auto Scaling** across two Availability Zones
- **Enable HTTPS** with a free certificate from AWS Certificate Manager
- **Attach an Elastic IP or use Route 53** so the address survives restarts
- **Right-size the instance** (t3.small or larger) instead of relying on swap
- **Store backups in S3** with versioning and lifecycle rules
- **Rebuild everything as code** with Terraform or CloudFormation

---

## Repository structure

```
.
├── README.md                  ← this storyline
├── architecture.svg           ← architecture diagram
├── data-dump-and-load.svg     ← backup and restore flow
└── 01-…png to 32-…png         ← 32 numbered screenshots in build order
```

**Deliberately not committed** (blocked by `.gitignore`):
- `*.pem` private keys, since anyone with the key could log in to the server
- `*.sql` database dumps, which contain the WordPress admin email and password hash

Screenshots have the AWS account ID, home IP address, and email redacted. The EC2 instance was terminated at the end of the project, so the IP addresses shown no longer point to anything.

---

*Built by B. Miller as part of the 1% University Cloud Engineering program.*
