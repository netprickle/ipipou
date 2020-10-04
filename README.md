# ipipou

Lightweight non-encrypted low-overhead L3 IPv4 tunnel natively supported by Linux.

The name is an acronym of IPIP-over-UDP which is effectively IPIP-over-FOU, where FOU is Foo-over-UDP.

`ipipou` utility helps to create such tunnels and adds optional remote side authentication using known remote IP:port and/or one of supported auth schemes:

- plaintext token

- public/verify key of the remote side (only ed25519 supported so far) and optional shared secret


## Advantages

IPIP-over-UDP in general and this tool in particular has some advantages over pure IPIP tunnel:

- In the global net tangible part of hardware optimized for 3 most popular protocols (TCP, UDP, ICMP), so other IP based protocols may have worse performance or disallowed at all. Being encapsulated in UDP IPIP tunnel going to be more stable and often faster.

- It's possible to create such tunnel even if some host is behind NAT (pure IPIP requires public IPs on both sides). Sometimes it's possible to create it when both sides are behind NAT (using stun and other techniques: not supported by this tool yet).

- Authentication support for remote IP:port connection. It's useful if one of hosts is behind NAT.

- Multiple tunnels can be created between the same public IPs pair (e.g. when multiple clients share the same public IP). To support it this implementation relies on local NAT configuration using additional private IP. In the future Linux may start to support [creating FOU tunnels to the same destination IP but different port](https://www.mail-archive.com/netdev@vger.kernel.org/msg228687.html) natively without this NAT hack, or may not...

- Live roaming support. Client side of connection may change its public IP and/or port, after [re]authentication packet tunnel will be reconfigured to use updated IP:port (WireGuard behaves similar way).

In comparison with encrypted tunnels (like WireGuard) IPIP-over-UDP has lower overhead and going to have higher throughput, lower latency and CPU usage, while still supports weak authentication. It can be better choice than WireGuard when tunnel level encryption is not required and better to be avoided, e.g. if inner layer going to be encrypted itself (https, ssh, vpn, etc.), or you prefer performance over security.

Nevertheless as opposed to WireGuard FOU supported by Linux only AFAIK.


## Requirements

Linux4.15+ (not tested with lower versions, 5.4+ recommended) built with support of IPIP and FOU, nftables, netfilter queue, python3.6+ (not tested with lower versions).

Internally `ipipou` tool relies on the following binaries: `ip`, `nft` for server mode, optionally `modprobe` and `conntrack`

External python3 libraries will be required:

- `netfilterqueue` for server mode
- `nacl` when ed25519 cryptography used for auth

Some checks relies on procfs and sysfs but fallback to other ways if does not exist.

`ipipou` must be run with root privileges OR have the following capabilities:
- CAP_NET_ADMIN - to create and configure network interfaces
- CAP_NET_RAW - if authentication packets sending required
- CAP_SYS_MODULE - if "fou" and "ipip" modules are not loaded yet


## Installation

Example for Debian/Ubuntu.

Download `ipipou` script and put to desired directory, e.g. to `/usr/local/bin/`.

Run all commands in root terminal.

For client mode with IP:port or token auth all requirements should be already satisfied, but in case if not:
```bash
apt install iproute2 python3
```

For client mode with AUTH_KEY auth `nacl` python library is required additionally:
```bash
apt install python3-nacl
```

For complete client/server mode:
```bash
apt install iproute2 python3 python3-nacl nftables conntrack build-essential libnfnetlink-dev libnetfilter-queue-dev python3-dev python3-pip
# There is no official deb package for NetfilterQueue, so use pip:
pip3 install -U NetfilterQueue -t /usr/local/lib/ipipou
# OR for newer version (you may try it if previous command failed):
pip3 install -U git+https://github.com/kti/python-netfilterqueue -t /usr/local/lib/ipipou
```


## Usage examples

### Command line

If you run in server mode and installed netfilterqueue to separate directory by `pip3 -t` (to not soil OS by systemwide pip) you have to change PYTHONPATH first (valid only for current shell session and its descendants):
```bash
export PYTHONPATH="/usr/local/lib/ipipou${PYTHONPATH:+:${PYTHONPATH}}"
```
Run `ipipou -h` for inline help.

Generate ed25519 key pair and share pubkey (2nd line) and shared secret ("topsecret" in this example) with the server
```bash
ipipou --auth-keygen -v
```

on server side:
```bash
PYTHONPATH="/usr/local/lib/ipipou${PYTHONPATH:+:${PYTHONPATH}}" \
ipipou -s -vvv -b @eth0 -n0 --auth-secret topsecret --auth-remote-pubkey-b64 2ndlinepublicbase64key
```

on client side:
```bash
ipipou -c -vvv -b @wlan0 -r 203.0.113.1:10000 --auth-key-b64 1stlineprivatebase64key --auth-secret topsecret --tunl-ip 172.28.0.1 --keepalive 27
```

On separate client terminal verify that the tunnel works
```
ping -c2 -w3 172.28.0.0  # Server side tunnel IP
```

When public IP or port changed (e.g. connection expired and NAT mapping changed) send SIGUSR1 to the process by `kill -SIGUSR1 PIDofIPIPOU`, or run the same command but with additional `--reauth-only` option.
To prevent connection expiry use `--keepalive SEC` option.

Full cleanup when all tunnels are done (will delete all ipip interfaces and fou listeners):
```bash
modprobe -r fou ipip  # Unload kernel modules
```

If client process dies (e.g. you kill it by `kill -9 PIDofIPIPOU`) the tunnel still remains in configured state and should work while connection is not expired, but on graceful exit (by ctrl+c or SIGTERM) the tunnel will be deconfigured first.


### As systemd service

- Put `ipipou@.service` to `/etc/systemd/system/` (or create symlink) and run `systemctl daemon-reload`.

- Create `NAME.conf` file(s) in `/etc/ipipou/` where `NAME` is arbitrary string. For file content syntax see `ipipou --help` for `--config` option.

    Config examples:

    `/etc/ipipou/server0.conf`:
    ```
    server
    number 0
    fou-dev eth0
    fou-local-port 10000
    tunl-ip 172.28.0.0
    auth-remote-pubkey-b64 eQYNhD/Xwl6Zaq+z3QXDzNI77x8CEKqY1n5kt9bKeEI=
    auth-secret topsecret
    auth-lifetime 3600
    reply-on-auth-ok
    verb 3
    ```
    `/etc/ipipou/client0.conf`:
    ```
    client
    number 0
    fou-local @wlan0
    fou-remote ipipou.example.net:10000
    tunl-ip 172.28.0.1
    # pubkey of auth-key-b64: eQYNhD/Xwl6Zaq+z3QXDzNI77x8CEKqY1n5kt9bKeEI=
    auth-key-b64 RuBZkT23na2Q4QH1xfmZCfRgSgPt5s362UPAFbecTso=
    auth-secret topsecret
    keepalive 27
    verb 2
    ```

    If config has sensitive info (like private keys), it's recommended to create `ipipou` user and group, set them for `ipipou@.service`, change permissions:
    ```
    adduser --system --no-create-home --group ipipou
    chown -RH 0:ipipou /etc/ipipou
    chmod 640 /etc/ipipou/*.conf
    # In ipipou@.service uncomment User=/Group= lines, comment DynamicUser=, then
    systemctl daemon-reload
    ```

- Run service as `systemctl start ipipou@NAME.service`


## Troubleshooting

- If ipipou service in server mode crashed next run may fail on first auth packet receiving (because previous configuration was not cleared). Additional restart may help (previous configuration will be cleared on clean stop).

- On client side in case if tunnel stops working (e.g. connection is expired somewhere in the middle, or you got new public ip), you can reload service (or send SIGUSR1) to send auth packet again and reestablish connection: `systemctl reload ipipou@NAME.service`

- If there was no traffic in tunnel for some period (so connection was expired) the client behind NAT going to be not accessible from server side. As a workaround you may
    - send keepalive packets regularly, e.g. every 27s, to either outer and/or inner tunnel layer (like auth and/or remote tunnel IP ICMP/ping packets correspondingly)
    - monitor connection and reauthenticate on failure (by the service reload or SIGUSR1 on client side)
    - run client in monitor mode (not supported yet)

- If you want to cleanup your system from all interfaces created by `ipipou` (like `tunl0`, or `ipipou*` in case if automatic cleanup failed) you may run `modprobe -r fou ipip; nft delete table ip ipipou`

- `ipipou` configuration of `nftables` should be fully compatible with your existing `iptables` configuration, `nftables` and `iptables` configurations can coexist. But in case if you use separate service for `nftables` managing, on service restart/reload `ip ipipou` table might be flushed, so the script may turn to inconsistent state. Be sure to keep `ip ipipou` table intact. It affects only server mode.


In case if you found a bug or have reasonable feature request feel free to create an issue or PR.
