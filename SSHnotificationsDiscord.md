In order to be pinged in Discord when a notification happens, you will need to know your own longDiscordID. Check this page on how to obtain that:
https://support.discord.com/hc/en-us/articles/206346498-Where-can-I-find-my-User-Server-Message-ID-

To get a channel webhook, check this linkg, 2nd Pragraph (Making a webhook):
https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks

Edit ''sshd'' config to send a log in/out message to discord:
```bash
cat <<EOF > /sbin/sshd-login
#!/bin/bash -x
# add to /sbin/ and make executable
# edit /etc/pam.d/sshd and add:
# session   optional   pam_exec.so /sbin/sshd-login
# to bottom of the file

WEBHOOK_URL="https://discord.com/api/webhooks/<your own discord hook>"
DISCORDUSER="<@longDiscordID>"

# Capture only open and close sessions.
case "\$PAM_TYPE" in
    open_session)
        PAYLOAD=" { \"content\": \"\$DISCORDUSER: User \\\`\$PAM_USER\\\` logged in to \\\`\$HOSTNAME\\\` (remote host: \$PAM_RHOST).\" }"
        ;;
    close_session)
        PAYLOAD=" { \"content\": \"\$DISCORDUSER: User \\\`\$PAM_USER\\\` logged out of \\\`\$HOSTNAME\\\` (remote host: \$PAM_RHOST).\" }"
        ;;
esac

# If payload exists fire webhook
if [ -n "\$PAYLOAD" ] ; then
    curl -X POST -H 'Content-Type: application/json' -d "\$PAYLOAD" "\$WEBHOOK_URL" &
fi
EOF
```
and make the file executable:
```bash
chmod +x /sbin/sshd-login
chown root:root /sbin/sshd-login
```
Add the sshd-login to execute at SSH log in/out:
```bash
cat << EOF >> /etc/pam.d/sshd

# Send login/logout messages to Discord
session   optional   pam_exec.so /sbin/sshd-login
EOF
```
And last, but not least: restart the sshd service:
```bash
systemctl restart sshd
```
