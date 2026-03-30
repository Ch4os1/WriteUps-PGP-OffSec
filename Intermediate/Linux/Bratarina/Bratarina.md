

### Lab Details 

- Difficulty: Intermediate
- Type: SMTP, Linux

#### Enumeration
- run `nmap`
```
PORT    STATE  SERVICE     VERSION
22/tcp  open   ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 db:dd:2c:ea:2f:85:c5:89:bc:fc:e9:a3:38:f0:d7:50 (RSA)
|   256 e3:b7:65:c2:a7:8e:45:29:bb:62:ec:30:1a:eb:ed:6d (ECDSA)
|_  256 d5:5b:79:5b:ce:48:d8:57:46:db:59:4f:cd:45:5d:ef (ED25519)
25/tcp  open   smtp        OpenSMTPD
| smtp-commands: bratarina Hello nmap.scanme.org [192.168.49.67], pleased to meet you, 8BITMIME, ENHANCEDSTATUSCODES, SIZE 36700160, DSN, HELP
|_ 2.0.0 This is OpenSMTPD 2.0.0 To report bugs in the implementation, please contact bugs@openbsd.org 2.0.0 with full details 2.0.0 End of HELP info
53/tcp  closed domain
80/tcp  open   http        nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title:         Page not found - FlaskBB
445/tcp open   netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: COFFEECORP)
```
#### Initial Foothold 
- `HTTP` port 80 runs a broken instance of `FlaskBB`
- `SMTP` is running on the target search online found an exploit for RCE https://www.exploit-db.com/exploits/48051
- Download the exploit and run against the target
- **NOTE**: use port 80 for reverse shell and ensure to use the `Debain` payload at the every bottom of the file 
```pl
## change lport
$lport = 4444;

## change payload to Debian
<SNIP>
} else { # RCE
	exec "nc -vl $lport" or die "Error: unable to execute netcat\n"; # BSD netcat
	#exec "nc -vlp $lport" or die "Error: unable to execute netcat\n"; # Debian netcat
	<SNIP>
}
```
- run the exploit
![[Pasted image 20260330230008.png]]
![[Pasted image 20260330230144.png]]
- obtained shell as root
#### Lateral Movement (If any)

#### Privilege Escalation
#### Resources

#### Lesson Learned
