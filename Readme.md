# HTTP Server, YUM Repository & DNS Server Setup (RHEL)

A step-by-step lab guide for configuring a **local YUM repository over HTTP**, a **DNS server (BIND)**, and an **Apache web server** on RHEL, using a server/client pair.

Based on the presentation *HTTP.pptx*.

---

## Table of Contents

1. [Lab Environment](#lab-environment)
2. [Background](#background)
3. [Part 1: YUM Repository + HTTP Server](#part-1-yum-repository--http-server)
4. [Part 2: DNS Server (BIND)](#part-2-dns-server-bind)
5. [Part 3: Apache Web Server (Virtual Host)](#part-3-apache-web-server-virtual-host)
6. [Troubleshooting & Corrections](#troubleshooting--corrections)

---

## Lab Environment

| Item | Value |
|---|---|
| OS | RHEL (ISO attached to the server VM) |
| Server IP | `192.168.1.10` |
| Client IP | `192.168.1.11` |
| Domain | `linux.com` |
| Package | `httpd` |
| Port | `80` (HTTP), `53` (DNS) |
| Config file | `/etc/httpd/conf/httpd.conf` |
| Daemon | `httpd` |

---

## Background

**HTTP** (Hypertext Transfer Protocol) is the foundation of the World Wide Web. It is an application-layer protocol used to transfer information between networked devices and runs on top of the other layers of the network protocol stack.

**YUM** (Yellowdog Updater, Modified) is a package manager developed at Duke University to improve installation of RPMs. It handles installing, updating and removing packages automatically, without updating each RPM by hand.

**DNS** (Domain Name System) translates domain names into IP addresses, so you can use `www.google.com` instead of remembering an IP address.

### DNS record types used

| Record | Purpose |
|---|---|
| **SOA** | Start of Authority. Holds meta information about the zone itself |
| **NS** | Points to a DNS name server for the zone |
| **A** | Maps a hostname to an IPv4 address |
| **CNAME** | Maps a hostname to another hostname (alias), e.g. `www` -> `server` |
| **MX** | Points to the SMTP mail server for the domain |
| **PTR** | Reverse lookup: maps an IP address to a hostname |

---

## Part 1: YUM Repository + HTTP Server

### Server side

```bash
# 1. Check that the ISO is attached
df -hT

# 2. Mount the ISO
mount /dev/sr0 /mnt

# 3. Create the directory that will serve packages over HTTP
mkdir -p /var/www/html/rhel

# 4. Copy BaseOS and AppStream from the ISO
cd /mnt
cp -r BaseOS AppStream /var/www/html/rhel
cd

# 5. Create the repo file
vim /etc/yum.repos.d/server.repo
```

`/etc/yum.repos.d/server.repo`:

```ini
[BaseOS]
name=BaseOS
baseurl=file:///var/www/html/rhel/BaseOS
gpgcheck=0
enabled=1

[AppStream]
name=AppStream
baseurl=file:///var/www/html/rhel/AppStream
gpgcheck=0
enabled=1
```

Verify the repo, then install and start Apache:

```bash
yum repolist all -v            # verify the repo configuration
yum install httpd              # press 'y' when prompted
systemctl restart httpd && systemctl enable httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
restorecon -Rv /var/www/html   # restore SELinux contexts
```

### Client side

```bash
vim /etc/yum.repos.d/base.repo
```

`/etc/yum.repos.d/base.repo`:

```ini
[BaseOS]
name=BaseOS
baseurl=http://192.168.1.10/rhel/BaseOS
gpgcheck=0
enabled=1

[AppStream]
name=AppStream
baseurl=http://192.168.1.10/rhel/AppStream
gpgcheck=0
enabled=1
```

```bash
yum repolist all -v
yum install httpd -y
```

> **Note:** The client must use an `http://` URL pointing at the server. A `file://` path would look for the packages on the client's own disk.

---

## Part 2: DNS Server (BIND)

### Server side

```bash
yum install bind -y
vim /etc/named.conf
```

In `options { ... }`, set:

```
listen-on port 53 { 192.168.1.10; };
allow-query     { any; };
```

Add the zones at the end of the file:

```
zone "linux.com" {
    type master;
    file "forward.zone";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "reverse.zone";
};
```

Create the zone files from the templates:

```bash
cd /var/named
ls
cp -vp named.localhost forward.zone
cp -vp named.loopback  reverse.zone
ls
```

**`/var/named/forward.zone`**

```
$TTL 1D
@   IN SOA  server.linux.com. root.server.linux.com. (
            0       ; serial
            1D      ; refresh
            1H      ; retry
            1W      ; expire
            3H )    ; minimum
        IN NS     server.linux.com.
server  IN A      192.168.1.10
@       IN A      192.168.1.10
client  IN A      192.168.1.11
www     IN CNAME  server.linux.com.
@       IN MX 10  server.linux.com.
```

**`/var/named/reverse.zone`**

```
$TTL 1D
@   IN SOA  server.linux.com. root.server.linux.com. (
            0       ; serial
            1D      ; refresh
            1H      ; retry
            1W      ; expire
            3H )    ; minimum
        IN NS   server.linux.com.
10      IN PTR  server.linux.com.
11      IN PTR  client.linux.com.
```

Point the server at itself, then start BIND:

```bash
vim /etc/resolv.conf
```

```
search linux.com
nameserver 192.168.1.10
```

```bash
systemctl restart named && systemctl enable named
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

Test:

```bash
nslookup server.linux.com      # or: dig server.linux.com
nslookup 192.168.1.10          # or: dig -x 192.168.1.10
```

### Client side

```bash
vim /etc/resolv.conf
```

```
search linux.com
nameserver 192.168.1.10
```

```bash
nslookup server.linux.com
nslookup 192.168.1.10
```

---

## Part 3: Apache Web Server (Virtual Host)

### Server side

```bash
yum install httpd -y
vim /etc/httpd/conf/httpd.conf
```

Go to the last line and add:

```apache
<VirtualHost *:80>
    ServerAdmin root@192.168.1.10
    DocumentRoot /var/www/html
    ServerName 192.168.1.10
    ErrorLog logs/server-error_log
    CustomLog logs/server-access_log common
</VirtualHost>
```

Check the syntax, create a test page, and start the service:

```bash
httpd -t
echo "welcome to web server" > /var/www/html/index.html
systemctl restart httpd && systemctl enable httpd
```

To write a custom page, edit `/var/www/html/index.html`.

### Client side

```bash
firefox 192.168.1.10
```

You should see: **welcome to web server**

---

## Troubleshooting & Corrections

Typos in the original slides that are corrected in this guide:

| Slide text | Correct |
|---|---|
| `systemctl enable http` | `systemctl enable httpd` |
| `--permenant` | `--permanent` |
| `firewall -cmd`, `firewall=-cmd` | `firewall-cmd` |
| `restorecon -rv` | `restorecon -Rv` |
| `yum repo list all` | `yum repolist all` |
| `yum install http`, `http -t` | `yum install httpd`, `httpd -t` |
| `index/html` | `index.html` |
| `/etc/reslov.conf` | `/etc/resolv.conf` |
| `Allow-qurry` | `allow-query` |
| `reversed.zone` / `reverse.zone` | use `reverse.zone` consistently |
| `customlog log/... commom` | `CustomLog logs/... common` |
| `nslookup 19.168.1.10` (client) | `nslookup 192.168.1.10` |

Common issues:

- **`named` fails to start:** run `named-checkconf` and `named-checkzone linux.com /var/named/forward.zone`. Zone files must be readable by the `named` group (`chgrp named /var/named/*.zone`).
- **Client can't reach the server:** confirm the firewall rules with `firewall-cmd --list-all` and that both machines are on the same subnet.
- **403 Forbidden in the browser:** re-run `restorecon -Rv /var/www/html` and check file permissions.
- **Repo not found on the client:** confirm `http://192.168.1.10/rhel/BaseOS/` opens in a browser or with `curl`.