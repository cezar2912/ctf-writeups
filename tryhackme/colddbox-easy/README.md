# ColddBox: Easy — TryHackMe

**Platform:** [TryHackMe](https://tryhackme.com/room/colddboxeasy)  
**Difficulty:** Easy  
**Focus:** WordPress enumeration, credential attack, authenticated PHP upload, credential reuse, Linux privilege escalation  
**Room author:** Marti from Hixec

## Attack path

1. Enumerate a WordPress site and discover `/hidden/`.
2. Confirm candidate WordPress usernames using different login error responses.
3. Find a weak administrator password with a wordlist attack.
4. Reach PHP execution through the administrator plugin upload flow, then obtain a shell as `www-data`.
5. Read a database credential from `wp-config.php` and use the reused password for SSH as `c0ldd`.
6. Identify a permissive `sudo` rule and escape from Vim into a root shell.

## 1. Reconnaissance

I scanned all TCP ports, requesting service versions and default scripts:

```bash
TARGET=<lab-ip>
nmap -Pn -sV -sC -p- --open "$TARGET"
```

The scan found HTTP on port **80** and SSH on the nonstandard port **4512**. The HTTP response advertised WordPress **4.1.31**. That version was a lead for later investigation; a version string alone does not confirm that a particular exploit works.

![Nmap services and WordPress generator](tryhackme/colddbox-easy/images/nmap.png)

I used `feroxbuster` to look for accessible web paths:

```bash
feroxbuster -u "http://$TARGET/" \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Most results were routine WordPress paths. `/hidden/` contained a message addressed to **C0ldd** about **Hugo** uploading articles, signed by **Philip**. I treated the names as possible account names, not proof that all three could log in.

![The message at /hidden/](hidden-message.png)

## 2. WordPress account discovery

At `/wp-login.php`, a bad password for `Hugo` produced a password-specific error, whereas an invented username produced an _invalid username_ error. I tested `C0ldd` and `Philip` the same way; both also received the password-specific response. This indicated those names were recognized by the WordPress login form.

![Known username receives a password error](known-user.png)

![Invented username receives an invalid username error](unknown-user.png)

I then used Hydra against the WordPress login form with `rockyou.txt`. The failure condition matched the password-specific error observed manually:

```bash
hydra -l c0ldd -P /usr/share/wordlists/rockyou.txt "$TARGET" \
  http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1:F=The password you entered for the username"
```

Hydra reported a credential for `c0ldd`. I verified it by signing in to WordPress and checking **Users → All Users**, where `c0ldd` had the **Administrator** role.

![WordPress user list showing the administrator role](wp-users.png)

## 3. Administrator access to a web shell

With administrator access, I explored the plugin upload workflow. WordPress rejected my PHP reverse shell as a plugin, but the file remained reachable under `/wp-content/uploads/`. Requesting its URL while a listener was running gave me a shell as `www-data`:

```bash
nc -lvnp 4444
# In the browser, request the uploaded PHP file at its observed URL.
```

This behavior is **consistent with** the PHP upload bypass described for older WordPress versions, including [CVE-2024-31210](https://www.cve.org/CVERecord?id=CVE-2024-31210). The advisory describes specific conditions around the plugin installer and temporary uploads. The observed outcome is the important finding here: an administrator-controlled PHP file executed from the uploads directory.

I upgraded the basic shell to make interaction easier:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Suspend the local netcat process with Ctrl+Z, then on my terminal:
stty raw -echo; fg
# Press Enter; in the remote shell:
stty rows 40 cols 130
id
```

The `id` result showed `uid=33(www-data)`

![Interactive shell running as www-data](www-data-shell.png)

## 4. Credential reuse and SSH

From the web shell, I checked the WordPress configuration file at `/var/www/wp-config.php`. Its `DB_PASSWORD` value was also accepted for the system account `c0ldd` over SSH on port 4512:

```bash
ssh -p 4512 c0ldd@"$TARGET"
```

Reusing a database secret as a system login password let me move from the low-privilege web account to an interactive user account. I found `user.txt` in the user's home directory.

## 5. Privilege escalation

As `c0ldd`, I checked the available `sudo` commands:

```bash
sudo -l
```

The output allowed `/usr/bin/vim`, `/bin/chmod`, and `/usr/bin/ftp` as root. I chose Vim because it can start a shell, and a Vim process launched through this rule runs as root:

![Sudo permissions for c0ldd](sudo-list.png)

```bash
sudo /usr/bin/vim
```

Inside Vim, I entered:

```vim
:set shell=/bin/sh
:shell
```

![root's id and home directory](root-id.png)

I could access `/root/root.txt`. This is an unsafe `sudo` permission for an interactive editor, not a flaw in Vim itself. [GTFOBins documents the Vim shell behavior](https://gtfobins.org/gtfobins/vim/).

## Findings and remediation

| Finding                                                           | What it enabled                                        | Defensive action                                                                                                                 |
| ----------------------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Distinct WordPress login errors and a weak administrator password | Account discovery and administrator login              | Use unique, strong passwords and MFA; rate-limit login attempts and avoid revealing whether an account exists.                   |
| PHP executable from a WordPress upload path                       | Code execution as `www-data` after administrator login | Update WordPress to a maintained release, restrict upload/install capabilities, and prevent PHP execution in upload directories. |
| Database password reused for SSH                                  | Access to the `c0ldd` system account                   | Use separate secrets for database and SSH accounts; rotate exposed credentials and restrict access to configuration files.       |
| Vim permitted with root `sudo` rights                             | Root shell from an interactive editor                  | Remove broad `sudo` access to shell-capable tools and grant only the specific administrative operations needed.                  |

**Takeaway:** The path combined several weaknesses. The web enumeration identified an account to test; administrator access led to code execution; password reuse enabled an SSH login; and a broad `sudo` rule turned that user access into root.

### References

- [TryHackMe room: ColddBox: Easy](https://tryhackme.com/room/colddboxeasy)
- [WordPress 6.4.3 security release](https://wordpress.org/news/2024/01/wordpress-6-4-3-maintenance-and-security-release/)
- [Wordfence: administrator PHP file upload](https://www.wordfence.com/threat-intel/vulnerabilities/wordpress-core/wordpress-core-497-php-file-upload?asset_slug=wordpress)
- [GTFOBins: Vim](https://gtfobins.org/gtfobins/vim/)
