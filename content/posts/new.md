---
title: "Windows, Linux and SSSD Woes!"
date: 2022-11-14T20:05:42+08:00
draft: false
tags: ["html", "css"]
categories: ["tech"]
---

Realmd and SSSD
 
Microsoft’s Active Directory or AD is the most popular Enterprise Access Management for decades now. So AD is a kind of distributed database, which is accessed remotely using the Lightweight Directory Access Protocol or popularly LDAP. 
But even though most organizations have a mix of Windows or Linux based environment, the authentication is normally centralized. So let’s see how we can add Linux Servers to an Active Directory Domain.
 
To integrate Linux servers with Microsoft Active Directory the main tool that you need is basically Realmd.
Realmd employs sssd (System Security Services Daemon) to do the actual lookups required for remote authentication and other heavy work of interacting with the domain.
Well, aside from realmd, there are a host of packages that need to be installed to make this work:
 
- sssd 
- realmd 
- oddjob 
- oddjob-mkhomedir 
- adcli 
- samba-common 
- samba-common-tools 
- krb5-workstation 
- openldap-clients 
- policycoreutils-python
 
Configuration
 
Make sure that both the AD system and the Linux system are properly configured, for example for a domain named `ad.example.com` :

- Verify the DNS SRV records: dig -t SRV _ldap._tcp.ad.example.com
- Verify AD records: dig -t SRV _ldap._tcp.dc._msdcs.ad.example.com
- Verify that system time on both systems is synchronized. This ensures that Kerberos is able to work properly.

These are the list of ports that need to be opened

| Service | Port | Protocol | Notes |
|---------|---------|---------|---------|
|DNS |53 |UDP and TCP| |
|LDAP|389|UDP and TCP|
|Kerberos | 88 | UDP and TCP|
|Kerberos | 464 | UDP and TCP| Used by kadmin for setting and changing a password |
| LDAP Global Catalog | 3268 |TCP |If the id_provider = ad option is being used |
| NTP | 123 | UDP |Optional|
 
Red Hat will recommend using the `realm join` command to configure the system.

```
sudo realm join --user=[domain user account] [domain name]
```
<br>
<br>

> You would probably need to use an account that has the privileges to join the server to the domain 
 
You can verify if the domain join was successful using the `realm list` command.
 
In other words, it is the primary interface between the directory service and the module requesting authentication services, `realmd`. Its main configuration file is located at `/etc/sssd/sssd.conf`. As a matter of fact, this is the main configuration file we will modify.
 

In one of my case I faced a weird issue where a server will not join to the domain; I went through a series of troubleshooting steps and here they are:
1. Set a hostname: sudo hostnamectl set-hostname <new_hostname>
2. Edit /etc/resolv.conf and set the correct DNS names
 
SSSD services and domains are configured in separate sections of this file, each beginning with a name of the section in square brackets. The following are examples:
```
[sssd]
[nss]
[pam] 
```

`[sssd]` → SSSD functionality is provided by specialized services that run together with SSSD. These specialized services are started and restarted by a special service called monitor. Monitor options and identity domains are configured in the [sssd] section of `/etc/sssd/sssd.conf`. The following is an example:
```
[sssd]
domains = LDAP
services = nss, pam
```

`[nss]` → Included in the sssd package is an NSS module, sssd_nss, which instructs the system to use SSSD to retrieve user information. This is configured in the [nss] section of /etc/sssd/sssd.conf
```
[nss]
filter_groups = root
filter_users = root
reconnection_retries = 3
entry_cache_timeout = 300
``` 

`[pam]` → The sssd package also provides a PAM module, sssd_pam, which is configured in the [pam] section of `/etc/sssd/sssd.conf`.
```
[pam]
reconnection_retries = 3
offline_credentials_expiration = 2
offline_failed_login_attempts = 3
offline_failed_login_delay = 5
```

The offline_credentials_expiration directive specifies, in days, how long to allow cached logins if the authentication provider is offline. The offline_failed_login_attempts directive specifies how many failed login attempts are allowed if the authentication provider is offline.
 
Update Configuration
 
To update the PAM configuration to reference all of the SSSD modules and to enable SSSD to create home directories use the authconfig command as follows to enable SSSD for system authentication:
```
authconfig --update --enablesssd --enablesssdauth --enablemkhomedir
```

Most probably you will also need make sure that sssd starts up automatically after a system reboot, so you would need to:
```
sudo systemctl enable sssd
sudo systemctl start sssd
sudo systemctl status sssd
```

Re-join a server back to domain
 
Well some times sh*t happens a server drops out of the domain and all kind of hell breaks lose. Normally just running a 
```
sudo systemctl status sssd
```
will show kerberos connectivity issues. So here’s a basic list of steps that I follow to have the server connect back to domain
 
1. Even though the server has dropped out from domain but sometimes the server itself doesn’t know that 😛 (Don’t ask me why!) So you will have to make sure the server understands that it needs to leave the domain
```
sudo realm leave <domain.com> -U <domain-admin>@<domain.com>
```
2. Now re-join back to the domain using:
```
sudo adcli -v join --domain-realm=<domain.com> --domain-controller=<domain-controller-server.domain.com> --login-user=<domain-admin>@<domain.com> -D <domain.com>
```
3. Re-install all the services once more
```
sudo yum install sssd sssd-tools realmd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation openldap-clients
sudo yum reinstall sssd-common
```
Update `/etc/sssd/sssd.conf` with the required config details of your domain
4. Enable and Start `sssd.conf`
sudo systemctl enable sssd
sudo systemctl start sssd
And voila! You should be good to go.
 
Reference:
1. Active Directory: https://www.redhat.com/sysadmin/linux-active-directory
2. Red Hat Documentation: https://www.thegeekdiary.com/understanding-system-security-services-daemon-sssd/
   


