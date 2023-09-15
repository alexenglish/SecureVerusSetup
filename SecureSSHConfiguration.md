## The Basics
*NOTE: I’ll refer to systems as “remote” and “local” throughout this document. When I say “remote” I’m meaning the system you want to access remotely. It should technically be local for the kind of setup we’re wanting to do but you do want to be able to access it remotely.*

With this guide you should be able to build a very secure access setup using SSH. 

### SSH Server Configuration
We'll make several basic changes, including changing the port number and disabling passwort authentication in favor of keys.

On most systems, the file you want to edit is located in: `/etc/ssh/sshd_config`

Find the `PasswordAuthentication` line and change it from "yes" to "no".

If you do not intend to log in as root (you can still configure to use sudo or su to escalate your privileges for administration) set the `PermitRootLogin` line to `no` and make sure it is uncommented. If you _do_ intend to login as root, you'll need to set this to `yes`, the other security measures will ensure this is secure. 

To access ssh on this host from the outside world you'll need to set up port forwarding (more on this later). We recommend doing this on a nonstandard port and forwarding to the correct port number for the ssh service running on this computer. You may leave it as the default 22, or may change it to a nonstandard port. This is unlikely to make a big difference in security, but some tools/scanners/malware/attackers may simply look for the default port 22 and move on if it isn't open. If you'd like to change the port, find the line with `Port` and change it to the port number you'd like. This is optional, just remember to use the correct port number when setting up port forwarding.

Run `systemctl restart sshd` to have your changes take effect without a reboot.

### Key generation
At the core level, you should _ALWAYS_ be using keys and not passwords. These keys should be encrypted and use ed25519 encryption. You'll need to create a key pair from each local system you would like to use to log into the remote system. That means each laptop, phone, etc., will have a key that will be installed on the system to enable login. These keys can be shared between devices, but this reduces flexibility in your options for disabling keys if they are compromised.

To generate a keypair on any *NIX based system: 
```
ssh-keygen -t ed25519
```

The first prompt asks you for the file, leave as default unless you want to call it something specific. 

Next it will ask for a passphrase, use something strong that you will remember and are not using for anything else. 

Afterwards you'll have a nice fresh SSH key to use to access your systems. These are located in your ~/.ssh folder. If you didnt name them something specific your private key will be called “id_ed25519” and your public key (the one you’ll be installing on systems to access over SSH) will be called “id_ed25519.pub”. Back up both securely, and only ever share the public key, and only share that with trusted parties that need to grant you access to systems.

When you use your key to log in you will be prompted for the password supplied when you generated it.

### Installing keys to authorize login
Lastly, on the remote system ensure your public key is installed for each user you'd like to be able to log in as.

Public keys are placed in .ssh/authorized_keys. If this directory and/or file do not exist you can create them with the following:
```
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod -R 700 .ssh
chmod -R 600 .ssh/authorized_keys
```

Now open up the authorized key file with your editor of choice and copy the contents of your local ~/.ssh/id_ed25519.pub (public key) file to the authorized_keys file on the first line (it should be all one line, though it may wrap depending on your editor and screen size). Any other keys you want to add for access will be on additional lines, one per line.

### 2FA via google-authenticator
Two-factor authentication is highly recommended. This would provide a code that can be used with google-authenticator, authy, numerous password managers, and other applications to generate one-time temporary passcodes that are required for login along with your normal credentials (your key)

0. Choose a 2FA application if you don't have one already. I’d' recommend Authy as it supports backup and recovery as well as blocking new devices. 
1. On the system you want to access remotely, find and install the libpam-google-authenticator equivalent for your distribution.
2. As the user you'll be logging into as run `google-authenticator`.
3. Follow the prompts ensuring you verify that your code is indeed working.
4. After getting your code set, you'll need to change two things.
  i. 
    Edit `/etc/pam.d/ssh`:
      - Under # Standard Un*x authentication comment out @include common auth and add under it:
        `auth required pam_google_authenticator.so`
      - This may be slightly different on other distributions but on debian-based that's what you need to change. 
  ii. 
    in your `/etc/ssh/sshd_config`
      - ensure that `ChallengeResponseAuthentication` and `UsePAM` are both set to "yes". In some systems `ChallengeResponseAuthentication` doesn't exist, in this case set `KbdInteractiveAuthentication` is set to "yes"
      - Check if there's already an `AuthenticationMethods` directive, if yes then comment it out.
      - At the bottom fo the file add the following:
        `AuthenticationMethods publickey,keyboard-interactive:pam`
    Then run `systemctl restart sshd` like before. 
5. In a new shell, try to SSH back in. You should have a request for a passcode, make sure your 2FA code works. 

Now you have a system with 2FA for SSH!

### Jump hosts
A jump host is an intermediary host for SSH access. All this does it make it so a nefarious actor cannot attempt to log in directly with just the IP address of their target system.

The jump host may be any host either on the local network (accessible from the outside via port forwarding) or can be external.

The setup for a jump host is the same as any Linux system, and they can be quite lightweight. It can also be the one with 2FA if you want (since your other system wont let any other hosts connect). We'll look at how connections are only permitted from jump hosts below in the firewall section.

On the jump host create a new user called `jump` and ensure the jump user has your SSH public key in it's authorized_keys file. This will allow you to log into the jump host, which can proxy your connection to the target system.

You may configure as many jump hosts as you like, and it is a good idea to have more than one so you don't lose access if one becomes unavailable.

You'll want to go through the ssd_conf setup steps done above, and it's highly recommended to choose a non-standard port number. You may also want to setup 2FA on this system.

To make using jump hosts manageable, you'll want to set up your `~/.ssh/config` file on each system you're connecting from.

Here is an example `~/.ssh/config`:
```
#jump0 jump
host jump0.j
    compression yes
    IdentitiesOnly yes
    IdentityFile ~/.ssh/id_ed25519
    User jump
    port 62328
    hostname x.x.x.z
    
Host myhost
    Compression yes
    IdentitiesOnly yes
    User root
    HostName x.y.x.y
    ProxyCommand /usr/bin/ssh -q -W %h:%p jump0.j
```
That config assumes that the ssh port of the jump host is set to 62328, and since no port is specified, it assumes port 22 on myhost that you're connecting to.

Now, when you run `ssh myhost`, it will use the entry for "myhost" which has a ProxyCommand that tells it to go through the jump host jump0.j.

To set up multiple jump hosts you can add them in the fashion of `jump0.j` above, then create dulicate entries for myhost with different names (e.g. myhost2) and the different jump host specified in the proxy command.

### Firewalls
Relevant to both jump hosts and remote hosts, you want to use a firewall. One recommended one is a software firewall called ufw. Install it how you normally install packages (if on debian/ubuntu, that’s just `apt install -y ufw`).

After installing ufw, you'll need to allow SSH access. Remember to use the default port number 22 if you didn't change that in your config, or use the non-standard port number you selected.

If you're not using a jump host:
```
ufw allow YOURSSHPORT/tcp
ufw enable
```
You'll want to do this on your jump host as well to block everything but SSH on the port that system is using.

If you're using a jump host you'll use this to permit incoming connections only from that host. You may permit more jump hosts by just repeating that top line with the IPs of the different jump hosts.
```
ufw allow from IP.OF.JUMP.HOST to any port YOURSSHPORT proto tcp
ufw enable
```

### Port forwarding
If you're on a private network (behind a router that issues private IP addresses) you'll need to enable port forwarding.

The exact process varies greatly between different router manufacturers. Generally speaking you'll log into your router and probably somewhere in advanced settings you'll look for `Port Forwarding`.

You'll want to add a rule/forward to the IP address of the system you'll connect to on your local network, on the port you configured for ssh there, and assign it a nonstandard port number for the external interface on the router.

So, say you configure it to forward port 99991 on your external interface (public IP) to your system at perhaps 192.168.1.55, on port 22 (if you didn't set a nonstandard port number) - that means:
* When you want to access your system from another system on the same private network, you'll access it as 192.168.1.55 on port 22.
* When you want to access your system from a system outside of your private network you'll access it with your network's public IP address on port 99991.

Make sure to set these port numbers and IPs accordingly in your `~/.ssh/config` file for easy access. If you have a laptop or cellphone that you'll use both on the private network and from other networks, you'll need to create multiple configurations then select the appropriate configuration for the connection you need.

### Dynamic DNS
On a typical residential connection your IP address is usually assigned dynamically, meaning it can change periodically, especially after a service outage, power interruption, or modem reset. There are a few ways to find your public IP address.

The admin interface or a status interface for your router may tell you. If you have the `dnsutils` package installed (or otherwise have the `dig` command), you can run `dig +short myip.opendns.com @resolver1.opendns.com` to get your public IP address. You can also go to whatsmyip.com or a number of other sites that want to advertise to you.

When you're connecting from outside the network this is the IP you'll use. But it's a good idea to set up a Dynamic DNS service so that when you're away from your network you have a way to discover the IP address in case it changes. Dynamic DNS usually means there is a service or scheduled task that runs on your computer, or a service you can configure in your router, that will keep a DNS service provider updated with your IP address so they can resolve a static URL to your network. In this way you can configure your ssh hosts in your configuration to point to that URL rather than an IP address and you should get connectivity even if your IP address has changed.

Check your router to see if it integrates with any services for this.

If you operate any domains check with your DNS provider to see if they have a DDNS solution you can utilize.

Failing that, see this list for providers that might suit your needs: https://www.comparitech.com/net-admin/dynamic-dns-providers/

The setup process will be different for every provider, so follow their documentation. When selecting providers make sure their solution is compatible with your operating system (a windows client won't help you on a Linux system).

### Mobile Apps
For Android I recommend (and use) JuiceSSH. For iOS, I use “SSH FIles - Secure Shellfish” as it was the only one that really supported jump hosts as I expected it to. Unfortunately it’s not free though.

In general you'll generate a key in the app. You'll need to transmit this key to yourself to put in the authorized_keys file of the system you're accessing, as well as on jump-hosts, if you're using them.

In JuiceSSH on Android if you tap on Manage Connections you can add hosts with lots of options provided via dropdowns. Configure jump hosts first and then when configuring the destination system(s) you can select the jump hosts as `Connect Via`. From the Manage Connections screen, if you swipe left (to go to the right) you'll be on the Identities screen where you can create ssh keys for authentication.
