# SSH Remote Server Setup guide

Solution for the [SSH Remote Server Setup](https://roadmap.sh/projects/ssh-remote-server-setup) project from roadmap.sh.

Set up a remote Linux server on Yandex Cloud and connect to it over SSH using two different key pairs, with convenient aliases in `~/.ssh/config`.

> **Note:** IP addresses, usernames and key names below are placeholders.

## 1. Create a VM in Yandex Cloud

I used **Yandex Cloud** (Compute Cloud). Other providers such as DigitalOcean or AWS work the same way, only the UI differs.

Configuration:

- **OS:** Ubuntu LTS
- **Public IP:** enabled
- **Security group:** inbound TCP port 22 allowed
- **User:** `<USER>`, with the **first public key** added in the "SSH key" field during VM creation

Resources:

- [Getting started: create a VM](https://yandex.cloud/ru/docs/getting-started/individuals/create-vm)
- [Connect to a Linux VM over SSH](https://yandex.cloud/ru/docs/compute/operations/vm-connect/ssh)

## 2. Generate two SSH key pairs

On the local machine:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/<KEY_NAME_1> -C "<COMMENT_1>"
ssh-keygen -t ed25519 -f ~/.ssh/<KEY_NAME_2> -C "<COMMENT_2>"
```

- `-f` sets the file path and name
- `-C` is an optional comment, not used for authentication

Each command creates a private key (`<KEY_NAME>`) and a public key (`<KEY_NAME>.pub`).

Show a public key (it is one line: `<type> <body> <comment>`):

```bash
cat ~/.ssh/<KEY_NAME_1>.pub
cat ~/.ssh/<KEY_NAME_2>.pub
```

Only public keys (`.pub`) are ever copied to the server. **Private keys stay on the local machine and are never committed to Git.**

## 3. Add both keys to the server

The first key was added when the VM was created (step 1). Connect with it first:

```bash
ssh -i ~/.ssh/<KEY_NAME_1> <USER>@<SERVER_IP>
```

On the first connection SSH shows a host authenticity warning. Compare the fingerprint with the one shown for the VM in the Yandex Cloud console, then type `yes`.

Now add the second public key on the server:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys     # paste the second public key on a new line, keep the existing one
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Each public key must be on its own line without line breaks inside it.

## 4. Test both keys

```bash
ssh -i ~/.ssh/<KEY_NAME_1> -o IdentitiesOnly=yes <USER>@<SERVER_IP>
ssh -i ~/.ssh/<KEY_NAME_2> -o IdentitiesOnly=yes <USER>@<SERVER_IP>
```

`IdentitiesOnly=yes` makes SSH use only the specified key, so it is clear which one actually works. Both connections succeed.

To end the SSH session and return to the local terminal, type `exit`:

## 5. SSH config aliases

On the local machine, in `~/.ssh/config`:

```sshconfig
Host <ALIAS_1>
    HostName <SERVER_IP>
    User <USER>
    IdentityFile ~/.ssh/<KEY_NAME_1>
    IdentitiesOnly yes

Host <ALIAS_2>
    HostName <SERVER_IP>
    User <USER>
    IdentityFile ~/.ssh/<KEY_NAME_2>
    IdentitiesOnly yes
```

Connect with the aliases:

```bash
ssh <ALIAS_1>
ssh <ALIAS_2>
```

## 6. Optional: connect with VS Code

1. Install [Visual Studio Code](https://code.visualstudio.com/) and the **Remote - SSH** extension.
2. Make sure the aliases from step 5 exist in `~/.ssh/config`.
3. Open the Command Palette (`Ctrl+Shift+P`) and run **Remote-SSH: Connect to Host...**.
4. Pick `<ALIAS_1>` or `<ALIAS_2>` and confirm the remote OS if asked.

Docs: [VS Code Remote - SSH](https://code.visualstudio.com/docs/remote/ssh).


## 7. Fail2ban

Fail2ban monitors authentication logs and temporarily bans IP addresses after too many failed SSH login attempts.

### Install Fail2ban

```bash
sudo apt update
sudo apt install -y fail2ban
```

### Configure SSH Protection

Fail2ban configuration files are stored in /etc/fail2ban/. You have two options: either create individual profiles for Fail2ban in multiple files within the jail.d/ directory, or create and collect all of your local settings in a jail.local file.

The jail.conf file will be periodically updated as Fail2ban itself is updated, and will be used as a source of default settings for which you have not created any overrides. Avoid editing jail.conf directly because package updates may overwrite your changes. Instead, create /etc/fail2ban/jail.local containing your custom settings:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add the following configuration:

```ini
[DEFAULT]
# Add your public IP to avoid banning yourself
ignoreip = 127.0.0.1/8 <YOUR_PUBLIC_IP>

[sshd]
enabled  = true
port     = ssh
backend  = systemd
maxretry = 3
findtime = 10m
bantime  = 1h
```

Replace `<YOUR_PUBLIC_IP>` with your actual public IP address. If you omit it, you may risk banning your own address after repeated failed attempts.

**Configuration options:**
- `ignoreip`: IP addresses that Fail2ban should never ban.
- `maxretry`: Number of failed attempts before a ban.
- `findtime`: Time window in which failed attempts are counted.
- `bantime`: How long an IP address remains banned.
- `backend = systemd`: Reads authentication logs from the systemd journal.

### Enable and Start Fail2ban

When you are finished editing, save and close the file. At this point, you can enable your Fail2ban service so that it will run automatically from ```--now``` on. 

First, run systemctl enable:

```bash
sudo systemctl enable fail2ban
```

Then, start it manually for the first time with systemctl start:

```bash
sudo systemctl start fail2ban
```

Restart the service after changing its configuration:

```bash
sudo systemctl restart fail2ban
```

### Verify the Configuration

Check that Fail2ban is running:

```bash
sudo systemctl status fail2ban
```

**Reference:** [How To Protect SSH with Fail2Ban on Ubuntu - DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04)