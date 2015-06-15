---
layout: post
title: Join an Active Directory domain from Linux
---

# Join an Active Directory domain from Linux

For this example I’m using “AD.EXAMPLE.COM”

## The really simple way...

Download [PowerBroker Identity Services Open](http://www.powerbrokeropen.org). This applies only if your distribution does not have it in a repository. It’ll either be a deb, an rpm or an sh (or rpm.sh). You may choose to buy the non-free version, it comes with GPO fittings for Linux and OSX machines (with the open version, group policy is not applied to joined non-Windows machines).

Then ensure the following are installed:

* samba
* smbldap-tools
* winbind
* openssl
* krb53-libs
* krb5-workstation (for kinit/klist)

They may not have those names on your distribution, but similar ones so YMMV.

You need to then install the PBIS Open package. The scariest part of the installation in my opinion is when it attempted to change SELinux modules in Red Hat and AppArmor modules in Ubuntu. If SELinux or AppArmor are disabled, they won’t be enabled. If they’re in permissive mode (otherwise known as monitor mode), they won’t be changed to enforce mode. If however you do have SELinux or AppArmor in enforce mode, YMMV and godspeed (in other words, yes SELinux is fantastic, but a nightmare to admin when you're not an actual sysadmin... get one handy if you're not one).

Once installation is complete, the next step is to then run (notice the hash):

``
# joindomain-cli join AD.EXAMPLE.COM username@AD.EXAMPLE.COM
Enter password:
``

This will attempt to join the computer to the domain. This can be finicky, so don’t be downtrodden if the output isn’t completely successful, you’re likely to have succeeded anyway.

Here is where I should give some common errors I got that *didn’t result* in success:

* No DNS route to a DC. One of the DCs will also likely be the DNS server, make sure the system can resolve it (edit /etc/resolv.conf if you need to manually include it).
* You’ve likely got the domain incorrect. I’ll admit this has burnt me more than once.

Whatever your output - unless it immediately returns failure or you’ve entered incorrect credentials - it’s possible that it actually worked.

Optionally, next up, check your kerberos settings. If your domain is part of a forest of domains, it’s likely that those were also returned. Technically this won’t have any bearing on the smooth operation, but I still like to limit the lines in my configuration files for ease of editing, and to not prod domains I don’t want to be part of.

Edit /etc/krb5.conf, find domains you’re not using and remove them. It’s that simple.

You can now test success by running “id” on a user in the domain:

`$ id AD\domainuser`

You should get a response back that looks a little like:

`uid=0000000000(AD\domainuser) gid=0000000000(AD\domain^users) groups=0000000000(AD\group_one),0000000000(AD\domain^users)`

It’ll look different (I’ve sanitised my results), and you may be a member of many groups. But still, huzzah! 

But you’ll fail to log in as a domain user just yet.

Before you get to actually logging in, try and see if you can get a kerberos ticket as a domain user:

``
$ kinit domainuser@AD.EXAMPLE.COM
Enter password:
``

If this works (check with `klist`) you’re almost all set for AD authentication!

Finally you’ll need to edit /etc/pam.d/password-auth and /etc/pam.d/system-auth as - depending on how the domain is set up - PBIS may assign requisites that aren’t actually needed (for example a smartcard).

Ensure the file has the following with its auth statements:

`auth sufficient pam_lsass.so try_first_pass`

Now you should be able to authenticate to AD (`pam_lsass.so` will assign kerberos tickets at login as well for both user and system, so anything that requires kerberos authentication will be done automatically).

Next, if you’re serious about using Windows AD users on a joined Unix system you’ll need to focus on two other things: sudoers and shells.

Sudoers is easy, just visudo and add AD\domainuser as the user entry giving them the sudo privileges they require (as opposed to ALL = (ALL) ALL or worse adding NOPASSWD).

The following three commands will allow you to set the domain by default so you don’t need to include AD\ in every command *and* set your default AD-user shell to bash. Naturally change /bin/bash to /usr/bin/zsh if you want.

``
/opt/pbis/bin/config UserDomainPrefix AD
/opt/pbis/bin/config AssumeDefaultDomain true
/opt/pbis/bin/config LoginShellTemplate /bin/bash
``

Something I haven’t mentioned: home directories.
PBIS will, by default, create users’ home directories in /home/local/AD/domainuser (AD and domainuser change based on domain and user name respectively). In order to give users home directories stored on a Windows file server, you’ll need to play with some samba magic. If it’s a \*nix server, nfs or whatever should be ok.
