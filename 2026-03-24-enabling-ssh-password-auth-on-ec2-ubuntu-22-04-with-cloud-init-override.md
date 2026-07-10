---
title: Enabling SSH Password Auth on EC2 (Ubuntu 22.04) with Cloud-Init Override
slug: enabling-ssh-password-auth-on-ec2-ubuntu-22-04-with-cloud-init-override
date_published: 2026-03-24T08:17:00.000Z
date_updated: 2026-03-24T08:22:52.000Z
excerpt: SSH still requires a public key even after enabling password auth? On EC2 Ubuntu, cloud-init might be silently overriding your config.
---

I ran into this while trying to send a large and confidential file from one EC2 to another. Both instances are running Ubuntu 22. The source EC2 is internal (company instance), no public IP, and only accessible via AWS Session Manager. Because of that, a lot of options are not viable.

S3 is not an option because I don’t want to configure AWS auth on that instance. It’s risky and might leave traces. FTP also doesn’t make sense here due to networking constraints. So the only realistic option left is `scp`. Normally I would do:

    scp -i /path/to/private-key.pem file.txt ubuntu@target-ip:/remote/path/
    

But I can’t bring the private key into the source EC2. Even if temporary, it can be logged and questioned later. So I need the target EC2 to accept SSH without key, meaning password-based authentication.

---

First attempt, modify `/etc/ssh/sshd_config` by setting `PasswordAuthentication yes` and `PubkeyAuthentication yes`, then restart using `sudo systemctl restart ssh`. After that, test the connection with `ssh ubuntu@target-ip`, which results in `Permission denied (publickey)`, meaning it still requires public key authentication.

Second attempt, disable public key authentication by setting `PasswordAuthentication yes` and `PubkeyAuthentication no`, then restart the SSH service again. After testing, the error changes to `Permission denied, please try again.`, which indicates that password authentication is now being attempted, but still failing.

---

After some digging, it became clear this wasn’t just a simple SSH config issue. On EC2 Ubuntu, cloud-init is involved in the initial VM setup and it can inject or override SSH configuration behind the scenes. Checking `ls /etc/ssh/sshd_config.d/` reveals an additional config file, `60-cloudimg-settings.conf`, which is not part of the main `sshd_config` but still gets loaded by SSH.

Looking into it with `grep -Ri passwordauthentication /etc/ssh/sshd_config.d/` shows `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf:PasswordAuthentication no`. At this point the behavior makes sense, because even though `PasswordAuthentication` was already set to `yes` in the main config, this cloud-init managed file is applied later and overrides it back to `no`.

---

The fix ends up being a single change on the cloud-init managed config. Update it directly with `sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf`, then verify using `grep -Ri passwordauthentication /etc/ssh/sshd_config.d/`, which should now return `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf:PasswordAuthentication yes`.

After that, restart SSH with `sudo systemctl restart ssh` and test again using `ssh ubuntu@target-ip`.

---

Environment for this write-up is Ubuntu 22.04.5 LTS using AMI ID `ami-07216ac99dc46a187`. This wasn’t tested on other Ubuntu versions, but should be reproducible in environments where cloud-init is involved.

> This is not something to leave enabled permanently. This is just a workaround to transfer files under constraints. After done, it should be reverted.
