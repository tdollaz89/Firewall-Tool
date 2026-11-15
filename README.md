# tunnel-fw

Builds and applies the nftables firewall for a two-hop tunnel setup:

```
clients ──client tunnel──▶ FE (ingress) ══transport tunnel══▶ GW (gateway)
                            DNAT + SNAT                clients terminate here
```

- **FE (Ingress):** Receives client connections and forwards their VPN traffic through the transport tunnel.
- **GW (Gateway):** Terminates transport tunnels and hosts the client VPN endpoint.

- **Client tunnel:** the tunnel clients build to the gateway. Its packets ride inside the transport tunnel.
- **Transport tunnel:** the FE ↔ gateway link that carries client traffic.

These are names for the two jobs, not for your interfaces. The tool detects or asks for the real interface names, whatever they are. The wizard also asks for each tunnel's own name (for example `river`, `salmon`). That name becomes the rule comment (`RIVER-VPN`, `SALMON-VPN`) and the transport tunnel's label in `doctor`, `list-fe` and `remove-fe`.

## The generated file

`/etc/nftables.conf` follows the hand-written house style:
- It starts with `flush ruleset`, with no header comments.
- One `inet filter` table, with chains in the order input, output, forward.
- `counter` on every accept rule.
- `<NAME>-VPN` comments on the tunnel rules.
- DHCP rules in and out, if you answer yes to the DHCP question.
- An outbound chain with the allow-list as sets (`udp dport { 53, 123 }`).
- A final `log prefix "nftables <chain> dropped: "` in each chain.

On an FE it also has a forward chain and an `ip nat` table with the DNAT/SNAT pair. The gateway never forwards, so it has no forward chain.

**Policy** is a required question: `accept` or `drop`, applied to input, output and forward.
- **`accept`:** unlisted traffic is logged and let through (log only). The file matches the hand-written style exactly.
- **`drop`:** unlisted traffic is blocked. The rules are the same as under `accept`.

- **Any tunnel combination.** Either tunnel can be WireGuard or OpenVPN: WireGuard over OpenVPN, OpenVPN over WireGuard, WireGuard over WireGuard, or OpenVPN over OpenVPN.
- **Either end can listen.** Either end of the transport tunnel can be the listening side: the FE, the gateway, or both.
- **Several FEs per gateway.** One gateway can be fed by several FEs, each with its own transport tunnel.

Install it on every server and run `tunnel-fw` on each. It reads the OpenVPN configs (`/etc/openvpn/**`), the WireGuard configs (`/etc/wireguard/*.conf` plus the live `wg` state) and the listening sockets, and lists what it finds at the top of the wizard for reference. The tunnel details themselves are typed in, not pre-filled. Then it asks which ports to keep open and applies the rules.

## Install

```sh
scp tunnel-fw <host>:/usr/local/sbin/ && ssh <host> chmod 755 /usr/local/sbin/tunnel-fw
```

Needs only Python 3 and `nft`. On FEs it offers to install `conntrack` if it's missing.

## Use

| Command | What it does |
|---|---|
| `tunnel-fw` (same as `tunnel-fw setup`) | Runs the wizard, saves `/etc/tunnel-fw/config.json`, then applies |
| `tunnel-fw --dry-run` | Asks the questions and prints the ruleset (syntax-checked as root); changes nothing |
| `tunnel-fw apply` | Re-applies the saved config, e.g. after editing the JSON |
| `tunnel-fw render` | Prints the ruleset the config produces |
| `tunnel-fw doctor [--fix]` | Checks boot persistence, each transport tunnel (interface, listener, WireGuard handshake, ping), FE NAT counters, stale NAT flows, and gateway clients per FE |
| `tunnel-fw add-fe` | **Gateway:** adds another FE's transport tunnel and re-applies |
| `tunnel-fw remove-fe NAME` | **Gateway:** removes an FE's transport tunnel, re-applies and cuts off its existing clients |
| `tunnel-fw list-fe` | Lists transport tunnels, whether each is up, and active clients per FE |

**No suggested answers for the tunnel details.** You type these every time:
- public interface
- client tunnel type, interface, port and subnet
- each transport tunnel's name, type, interface, port and listening side
- both tunnel IPs
- on a gateway, the FE's public IP
- on an FE, the ports to forward (at least one)

A blank answer is asked again. The rest (open ports, outbound allow-list, the yes/no questions) shows a default, taken from the saved config when you rerun the wizard. To change one thing without retyping everything, edit `/etc/tunnel-fw/config.json` and run `tunnel-fw apply`.

Port lists use the form `proto/port`, separated by commas: `tcp/22`, `tcp/8000-8100`, `tcp/22@203.0.113.0/24` (source-restricted). Forwards can remap a port with `tcp/443:1194` (public port 443 goes to gateway port 1194).

Open ports can carry a label, which becomes the rule's comment: `tcp/20022=CUSTOM-SSH-PORT, tcp/22`. Unlabelled `tcp/22` gets `SSH`. Labels already present in a server's current rules are kept as defaults.

## What the wizard asks

### On an FE

- **The transport tunnel:**
  - Name (for example `river`)
  - Type: `wireguard` or `openvpn`
  - Interface
  - Port, on the listening side. A bare number means UDP; write `tcp/443` for OpenVPN over TCP. WireGuard must be UDP.
  - Which side listens: `fe`, `gw` or `both`
  - Tunnel IPs at both ends

  The FE never asks for the gateway's public IP.
- **Forwards:** the client tunnel's public port(s), for example `udp/11223` for WireGuard or `tcp/1194` for OpenVPN.

### On a gateway

- **The client tunnel:**
  - Name (for example `salmon`), type, interface, `proto/port` and subnet
  - For OpenVPN, a status file so `doctor` can list clients
  - Whether to accept clients **only through the transport tunnels**. This hides the gateway from direct access.
- **The transport tunnels:** one per FE, with the same questions as on the FE side, plus the FE's public IP. It's required if the gateway connects out to the FE. If the gateway listens, it's optional and restricts who may connect.

### Both roles

- Public interface
- Whether the server gets its IP by DHCP. The default is what the tool detects. Yes adds `udp sport 67 udp dport 68` inbound and `udp sport 68 udp dport 67` outbound. Servers with static addresses don't need them. Under `drop`, a DHCP server without them loses its address when the lease runs out.
- Open services
- Allowed outbound ports (default `udp/53, udp/123, tcp/80, tcp/443`; the tunnels' own traffic is always added)
- Policy for unlisted traffic: `accept` (log only) or `drop`, no default
- Whether to log unlisted traffic

### What "who listens" does to the rules

| Side | Inbound | Outbound |
|---|---|---|
| Listening side | `proto dport PORT` (on a gateway, optionally only from the FE's public IP) | (nothing; replies are established traffic) |
| Connecting side | (nothing; replies are established traffic) | `proto dport PORT`; on a gateway, `ip daddr FE_PUBLIC proto dport PORT` |

## Adding FEs to an existing gateway

```
fe1 ══ transport tunnel (OpenVPN,   172.3.1.1 <-> 172.3.1.2) ══╗
fe2 ══ transport tunnel (WireGuard, 10.200.0.1 <-> 10.200.0.2) ══╬══▶ GW
fe3 ══ transport tunnel (OpenVPN,   172.3.3.1 <-> 172.3.3.2) ══╝
```

Transport tunnels on the same gateway can be different types. On day one, run `tunnel-fw` on the gateway with the first FE. To add fe2 later:

1. Bring up fe2's transport tunnel at both ends. Give it **its own tunnel IPs**; don't copy fe1's `ifconfig` or `Address` line.
2. On fe2, run `tunnel-fw` and choose role `fe`.
3. On the gateway, run `tunnel-fw add-fe`. It asks only about the new FE; your existing setup isn't asked about again. If it finds the new tunnel's config, it shows it for reference. To script it:
   ```sh
   tunnel-fw add-fe --name fe2 --type wireguard --dev wgfe2 --port udp/51820 \
       --listener gw --local-ip 10.200.0.2 --peer-ip 10.200.0.1 --fe-ip <fe2 public ip>
   ```
4. Run `tunnel-fw doctor` on the gateway. It shows which FE each client is coming through.

Why the tunnel IPs must be unique: each FE SNATs client traffic to its own transport tunnel IP. The gateway sends replies back by that address, so the address is what picks the right transport tunnel. The tool refuses duplicate names, interfaces, transport tunnel IPs, or two transport tunnels listening on the same port.

`remove-fe` won't remove the last transport tunnel. After removing one, stop its tunnel (`systemctl disable --now wg-quick@...` or `openvpn@...`).

Config files from older versions convert automatically when loaded. Those versions used `river`/`rivers` or `backhauls` for the transport tunnel, and `salmon` or `wg` for the client tunnel. Options removed in 1.6 (`trust_clients`, `route_clients`, `client_to_client`, and the gateway's public IP on an FE) are dropped from the config when it loads. Configs from before 1.6 have no `dhcp` setting and load with `"dhcp": true`, the old always-on behaviour. Set it to `false` and run `apply` to remove the rules.

## Safety

- **Validated before loading.** `nft -c` checks the ruleset first, and the load is atomic. If it fails, the live rules are left as they were.
- **Lockout guard.** If your current SSH port isn't in the open list, the wizard warns you and offers to add it.
- **Rollback timer.** After loading, you have 60 seconds (`--timeout`) to type `yes`. If you don't, or your SSH session drops, it restores the previous ruleset and `ip_forward` value. The timer runs in its own session, so it still fires if your SSH session dies. Skip it with `--no-rollback`.
- **Backups.** Before each apply it saves the live ruleset and the old `/etc/nftables.conf` to `/var/lib/tunnel-fw/` and keeps the last 10.
- **Warnings.** It tells you before removing nft tables it doesn't manage (for example docker's) and when ufw is active.

## The two problems it fixes permanently

1. **Rules not loaded at boot.** It enables `nftables.service`, which loads `/etc/nftables.conf` before networking comes up.
2. **Stale un-NATed conntrack flows on FEs.** If a client is already sending before the DNAT rule exists, conntrack records the flow without NAT. The client keeps retrying from the same source port, so the UDP timeout never expires and the client never connects. After every apply, the tool clears conntrack entries for the forwarded ports. `doctor` detects these flows and `doctor --fix` deletes them.

Note: on FEs only, `ip_forward` is turned on and persisted by setting `net.ipv4.ip_forward=1` in `/etc/sysctl.d/99-sysctl.conf` (`/etc/sysctl.conf`). The tool uncomments or edits an existing line rather than adding a file. The gateway's sysctl settings aren't touched. The `90-tunnel-fw.conf` file written by older versions is removed on the next apply.
