## Clam_AV

### Lab Details 

- Difficulty: Easy 
- Type: SMTP Injection

#### Enumeration
- run `nmap`
```
$ sudo nmap -p- 192.168.120.81
Starting Nmap 7.80 ( https://nmap.org ) at 2020-03-23 09:55 EDT
Nmap scan report for 192.168.120.81
Host is up (0.032s latency).
Not shown: 65528 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
139/tcp   open  netbios-ssn
199/tcp   open  smux
445/tcp   open  microsoft-ds
60000/tcp open  unknown
```
- found that the target is running `smtp`
- enumerate `smtp` with `smtp-check`
```
kali@kali:~$ snmp-check 192.168.120.94
snmp-check v1.9 - SNMP enumerator
Copyright (c) 2005-2015 by Matteo Cantoni (www.nothink.org)

[+] Try to connect to 192.168.120.94:161 using SNMPv1 and community 'public'

[*] System information:

  Host IP address               : 192.168.120.94
  Hostname                      : 0xbabe.local
  Description                   : Linux 0xbabe.local 2.6.8-4-386 #1 Wed Feb 20 06:15:54 UTC 2008 i686
  Contact                       : Root <root@localhost> (configure /etc/snmp/snmpd.local.conf)
  Location                      : Unknown (configure /etc/snmp/snmpd.local.conf)
  Uptime snmp                   : 00:33:28.71
  Uptime system                 : 00:32:46.43
  System date                   : 2020-4-28 16:22:09.0

...

  3761                  runnable              klogd                 /sbin/klogd                               
  3765                  runnable              clamd                 /usr/local/sbin/clamd                      
  3767                  runnable              clamav-milter         /usr/local/sbin/clamav-milter  --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl
  3776                  runnable              inetd                 /usr/sbin/inetd
...
```
#### Initial Foothold 

#### Lateral Movement (If any)

#### Privilege Escalation
- identified that the target is running `clamd` and `clamav-milter`
- search online found that its vulnerable to command injection
- download and exploited script: https://github.com/strikoder/sendmail-clamav-exploit-CVE-2007-4560/blob/main/sendmail_clamav_exploit.py
- obtained root shell

#### Resources

#### Lesson Learned
- SMTP enumeration 
