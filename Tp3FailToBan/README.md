# Module 7 : Fail2Ban

Fail2Ban c'est un peu le cas d'école de l'admin Linux, je vous laisse Google pour le mettre en place.

Faites en sorte que :

- si quelqu'un se plante 3 fois de password pour une co SSH en moins de 1 minute, il est ban
- vérifiez que ça fonctionne en vous faisant ban
- afficher la ligne dans le firewall qui met en place le ban
- lever le ban avec une commande liée à fail2ban

**Installation de Fail2Ban**

```bash
sudo dnf install epel-release
Complete!
```

```bash
sudo dnf install fail2ban fail2ban-firewalld
Installed:
  esmtp-1.2-19.el9.x86_64                 fail2ban-1.0.1-2.el9.noarch           fail2ban-firewalld-1.0.1-2.el9.noarch
  fail2ban-sendmail-1.0.1-2.el9.noarch    fail2ban-server-1.0.1-2.el9.noarch    libesmtp-1.0.6-24.el9.x86_64
  liblockfile-1.14-9.el9.x86_64           python3-systemd-234-18.el9.x86_64

Complete!
```

**On lance le service Fail2Ban**
```bash
sudo systemctl start fail2ban
sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; disabled; vendor preset: disabled)
     Active: active (running) since Mon 2022-11-21 11:20:36 CET; 7s ago
       Docs: man:fail2ban(1)
    Process: 1492 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 1493 (fail2ban-server)
      Tasks: 3 (limit: 5907)
     Memory: 12.3M
        CPU: 134ms
     CGroup: /system.slice/fail2ban.service
             └─1493 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 21 11:20:36 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 21 11:20:36 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 21 11:20:37 db.tp2.linux fail2ban-server[1493]: 2022-11-21 11:20:37,032 fail2ban.configreader   [1493]: WARNING 'al>
Nov 21 11:20:37 db.tp2.linux fail2ban-server[1493]: Server ready
```

```bash
sudo systemctl enable fail2ban
Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service → /usr/lib/systemd/system/fail2ban.service.
```

**On change la configuration de Fail2ban pour indiquer la durée du ban et le nombre de tentatives avant le ban :**
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local


sudo nano /etc/fail2ban/jail.local
[DEFAULT]
bantime = 1h
findtime = 1h
maxretry = 3


sudo mv /etc/fail2ban/jail.d/00-firewalld.conf /etc/fail2ban/jail.d/00-firewalld.local
```

**On restart le service pour appliquer les changements :**
```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; vendor preset: disabled)
     Active: active (running) since Mon 2022-11-21 11:31:06 CET; 3s ago
       Docs: man:fail2ban(1)
    Process: 1570 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 1571 (fail2ban-server)
      Tasks: 3 (limit: 5907)
     Memory: 10.6M
        CPU: 70ms
     CGroup: /system.slice/fail2ban.service
             └─1571 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 21 11:31:06 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 21 11:31:06 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 21 11:31:06 db.tp2.linux fail2ban-server[1571]: 2022-11-21 11:31:06,316 fail2ban.configreader   [1571]: WARNING 'al>
Nov 21 11:31:06 db.tp2.linux fail2ban-server[1571]: Server ready
```

**Configuration pour Fail2Ban pour les connexions SSH :**
```bash
sudo cat /etc/fail2ban/jail.d/sshd.local
[sshd]
enabled = true

# Override the default global configuration
# for specific jail sshd
bantime = 1d
maxretry = 3
```

**Et on redémarre pour appliquer la conf :**
```
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; vendor preset: disabled)
     Active: active (running) since Mon 2022-11-21 11:33:11 CET; 5s ago
       Docs: man:fail2ban(1)
    Process: 1591 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 1592 (fail2ban-server)
      Tasks: 5 (limit: 5907)
     Memory: 14.0M
        CPU: 144ms
     CGroup: /system.slice/fail2ban.service
             └─1592 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 21 11:33:16 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 21 11:33:16 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 21 11:33:16 db.tp2.linux fail2ban-server[1592]: 2022-11-21 11:33:16,668 fail2ban.configreader   [1592]: WARNING 'al>
Nov 21 11:33:16 db.tp2.linux fail2ban-server[1592]: Server ready
```


**On peut bien voir qu'on a une prison pour les connexions via SSHD :**
```
sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd
```

**On vérifie bien qu'on a 3 chances avant de se faire ban :**
```
sudo fail2ban-client get sshd maxretry
3
```

**On test maintenant pour voir si Fail2Ban marche en essayant de se connecter depuis la machine `web.tp2.linux` sur notre machine `db.tp3.linux` :**

```
ssh mehdi@10.102.1.12
mehdi@10.102.1.12's password:
Permission denied, please try again.
mehdi@10.102.1.12's password:
Permission denied, please try again.
mehdi@10.102.1.12's password:
mehdi@10.102.1.12: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
ssh mehdi@10.102.1.12
ssh: connect to host 10.102.1.12 port 22: Connection refused
```

**On peut bien voir qu'après avoir raté 3 fois, on ne peut même plus se connecter en SSH la connexion est refusée et si l'on retourne voir dans la prison SSHD, on peut voir l'addresse IP de la machine `web.tp2.linux` dans la liste des bannis :**
```
sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     3
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   10.102.1.11
```

**Ensuite il me reste juste a me déban au cas où parce que j'avais rien fait moi :'(**

```
sudo fail2ban-client unban 10.102.1.11
1
sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     3
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     1
   `- Banned IP list:
```



**Ensuite on refait cette installation sur toutes nos autres machines mais je pense qu'il n'y a pas besoin de le remettre dans le compte rendu.**
