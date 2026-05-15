Yes: **HTB + CAKE is a good combo** for WireGuard user quotas, but there is one important rule:

Do **not** set every CAKE leaf to the full NIC rate when the parent HTB class is enforcing a smaller limit. HTB will still cap the class, but CAKE's own shaping and fairness math will be wrong for that leaf. For per-user shaping, set each CAKE leaf to that user's effective cap.

Sources used:
- iproute2 `tc` docs via Context7 for qdisc/class/filter structure and handle usage
- `tc-cake` manpage for CAKE `bandwidth RATE` and `unlimited` behavior: https://manpages.ubuntu.com/manpages/noble/en/man8/tc-cake.8.html

**Recommended Model**

Use two tiers:

| Tier | Quota | Pool Ratio | On 1Gbps Server | On 10Gbps Server |
|---|---:|---:|---:|---:|
| Free | 5GB/month | 20% | 200Mbps | 2Gbps |
| Paid | unlimited | 80% | 800Mbps | 8Gbps |

If paid users should all have the same hard cap, such as `100mbit` each, do not rely only on CAKE host fairness at the tier level. If only one paid user is active, they can consume most of the paid pool. Add a per-peer HTB class for each user and place CAKE on that leaf.

Use this hierarchy:

```text
wg0 root HTB
├── free tier pool: 20% of server capacity
│   ├── free peer class: per-user cap
│   │   └── CAKE
│   └── free peer class: per-user cap
│       └── CAKE
└── paid tier pool: 80% of server capacity
    ├── paid peer class: per-user cap
    │   └── CAKE
    └── paid peer class: per-user cap
        └── CAKE
```

HTB enforces:
- total free pool
- total paid pool
- per-user max speed

CAKE handles:
- flow fairness inside a single user tunnel
- lower latency under load
- better queue behavior than FIFO

**Capacity Rule**

Use a shaped root slightly below real capacity. For a nominal 1Gbps VPS, start with `950mbit` or `980mbit`, not exactly `1000mbit`, because the real bottleneck may be lower than the advertised NIC speed.

Example:

```text
serverCapacityMbps = 950

freePool = 190Mbps
paidPool = 760Mbps
```

For cleaner product numbers, you can still store `1000`, but measured operational capacity is better if you want CAKE/HTB to control queueing accurately.

**Download Direction**

Traffic to clients leaves `wg0`, so match `dst peer_ip`.

Create the download-side root and tier pools:

```bash
tc qdisc replace dev wg0 root handle 1: htb default 999

tc class replace dev wg0 parent 1: classid 1:1 htb rate 950mbit ceil 950mbit

tc class replace dev wg0 parent 1:1 classid 1:10 htb rate 190mbit ceil 190mbit
tc class replace dev wg0 parent 1:1 classid 1:20 htb rate 760mbit ceil 760mbit
```

Paid peer capped at `100mbit`:

```bash
tc class replace dev wg0 parent 1:20 classid 1:2002 htb rate 10mbit ceil 100mbit
tc qdisc replace dev wg0 parent 1:2002 handle 2002: cake bandwidth 100mbit besteffort

tc filter replace dev wg0 protocol ip parent 1: prio 20 u32 \
  match ip dst 10.8.0.2/32 flowid 1:2002
```

Free peer capped at `10mbit`:

```bash
tc class replace dev wg0 parent 1:10 classid 1:1002 htb rate 1mbit ceil 10mbit
tc qdisc replace dev wg0 parent 1:1002 handle 1002: cake bandwidth 10mbit besteffort

tc filter replace dev wg0 protocol ip parent 1: prio 20 u32 \
  match ip dst 10.8.0.3/32 flowid 1:1002
```

**Upload Direction**

Traffic from clients arrives on `wg0` ingress. To shape it, redirect ingress to `ifb0`, then build a separate HTB tree on `ifb0`.

Step 1: create and enable `ifb0`.

```bash
modprobe ifb
ip link add ifb0 type ifb 2>/dev/null || true
ip link set ifb0 up
```

Step 2: redirect `wg0` ingress into `ifb0`.

```bash
tc qdisc replace dev wg0 ingress
tc filter replace dev wg0 parent ffff: protocol ip u32 match u32 0 0 \
  action mirred egress redirect dev ifb0
```

Step 3: build the upload-side HTB tree on `ifb0`.

```bash
tc qdisc replace dev ifb0 root handle 2: htb default 999

tc class replace dev ifb0 parent 2: classid 2:1 htb rate 950mbit ceil 950mbit

tc class replace dev ifb0 parent 2:1 classid 2:10 htb rate 190mbit ceil 190mbit
tc class replace dev ifb0 parent 2:1 classid 2:20 htb rate 760mbit ceil 760mbit
```

Step 4: add per-peer upload classes to `ifb0` and match `src peer_ip`.

Paid peer capped at `100mbit`:

```bash
tc class replace dev ifb0 parent 2:20 classid 2:2002 htb rate 10mbit ceil 100mbit
tc qdisc replace dev ifb0 parent 2:2002 handle 2002: cake bandwidth 100mbit besteffort

tc filter replace dev ifb0 protocol ip parent 2: prio 20 u32 \
  match ip src 10.8.0.2/32 flowid 2:2002
```

Free peer capped at `10mbit`:

```bash
tc class replace dev ifb0 parent 2:10 classid 2:1002 htb rate 1mbit ceil 10mbit
tc qdisc replace dev ifb0 parent 2:1002 handle 1002: cake bandwidth 10mbit besteffort

tc filter replace dev ifb0 protocol ip parent 2: prio 20 u32 \
  match ip src 10.8.0.3/32 flowid 2:1002
```

If `tc filter replace` on `ifb0` fails with:

```text
Error: Change operation not supported by specified qdisc.
```

delete and add the filter instead:

```bash
tc filter delete dev ifb0 parent 2: prio 20
tc filter add dev ifb0 protocol ip parent 2: prio 20 u32 \
  match ip src 10.8.0.3/32 flowid 2:1002
```

Use the same pattern for a paid peer by changing the source IP and `flowid`.

**Verify**
```sh
tc -s qdisc show dev wg0
tc -s qdisc show dev ifb0
tc class show dev wg0
tc class show dev ifb0
tc filter show dev wg0
tc filter show dev ifb0
```

When moving a peer between tiers, check for stale duplicate filters. An IP should map to only one class per direction. For example, do not leave the same peer in both `1:2002` and `1:1002` on `wg0`.

If needed, remove stale filters first and then add the intended mapping:

```bash
tc filter show dev wg0
tc filter show dev ifb0
```

**Important Handle Notes**

- `parent 2:` means the root qdisc with handle `2:` must already exist on `ifb0`
- redirecting packets to `ifb0` does **not** create that qdisc automatically
- `classid 2:2002` is the HTB class
- `handle 2002:` is the CAKE qdisc ID attached under that class
- a qdisc handle such as `22002:` is invalid; use values such as `2002:` or another unused valid handle

**Manual CLI Speed Test**

Use `iperf3` through the WireGuard tunnel. There are two valid test styles:

- test against the server VPN IP `10.8.0.1`
- test against a public IP and interpret direction correctly

Testing against `10.8.0.1` is the cleanest tunnel-only test, but it only works if the client `AllowedIPs` includes `10.8.0.1/32` or a wider range that covers it.

On the server:

```bash
iperf3 -s -B 10.8.0.1
```

On the free user device:

```bash
iperf3 -c 10.8.0.1 -t 20 -P 4
```

That tests client -> server and exercises the upload cap on `ifb0`.

To test the opposite direction:

```bash
iperf3 -c 10.8.0.1 -t 20 -P 4 -R
```

That tests server -> client and exercises the download cap on `wg0`.

For a free user capped at `10mbit`, expect roughly `9-10 Mbits/sec` in each direction.

If the client does not route `10.8.0.1` through the tunnel, you can still test using a public destination. In that case:

```bash
iperf3 -c PUBLIC_IP -t 20 -P 4
```

tests client -> server upload and exercises `ifb0`.

```bash
iperf3 -c PUBLIC_IP -t 20 -P 4 -R
```

tests server -> client download and exercises `wg0`.

This distinction matters when debugging: if download is limited correctly but upload is still full speed, inspect `ifb0`, not `wg0`.

While testing, inspect counters:

```bash
tc -s class show dev wg0
tc -s class show dev ifb0
tc filter show dev wg0 parent 1:
tc filter show dev ifb0 parent 2:
```

**Scalable Config Design**

Do not hard-code `200mbit` and `800mbit`.

Store server traffic policy in config or DB:

```ts
type ServerTrafficPolicy = {
  serverId: string
  interfaceName: string        // wg0
  ifbName: string              // ifb0
  capacityMbps: number         // 950, 10000, etc.
  freePoolRatio: number        // 0.20
  paidPoolRatio: number        // 0.80
  freeUserMbps: number | null  // null = derive from base
  paidUserMbps: number | null  // null = derive from base
  freeMonthlyQuotaBytes: number
}
```

Then compute:

```ts
const freePoolMbps = Math.floor(capacityMbps * freePoolRatio)
const paidPoolMbps = Math.floor(capacityMbps * paidPoolRatio)

const capacityScale = capacityMbps / 1000

const freeUserMbps =
  policy.freeUserMbps ?? Math.floor(10 * capacityScale)

const paidUserMbps =
  policy.paidUserMbps ?? Math.floor(100 * capacityScale)
```

That gives:

| Server | Capacity | Free Pool | Paid Pool | Free User | Paid User |
|---|---:|---:|---:|---:|---:|
| 1Gbps | 1000Mbps | 200Mbps | 800Mbps | 10Mbps | 100Mbps |
| 10Gbps | 10000Mbps | 2000Mbps | 8000Mbps | 100Mbps | 1000Mbps |

In practice, per-user caps should be overrideable per server. Pool scaling and user-cap scaling should be separate knobs.

**Backend Design**

Add a `traffic_policy` concept:

```ts
type Tier = 'free' | 'paid'

type PeerTrafficAssignment = {
  peerId: string
  userId: string
  tier: Tier
  assignedIp: string
  publicKey: string
  downloadClassId: string
  uploadClassId: string
  userLimitMbps: number
  monthlyQuotaBytes: number | null
}
```

Flow:

1. User connects.
2. Backend resolves tier:
   - active entitlement = `paid`
   - no entitlement = `free`
3. Backend checks free monthly quota before issuing config.
4. Backend registers peer.
5. Agent applies:
   - tier pool classes for server if missing
   - per-peer HTB class on `wg0`
   - per-peer HTB class on `ifb0`
   - CAKE leaf with `bandwidth userLimitMbps`
   - filters by assigned peer IP
6. Agent polls WireGuard transfer counters.
7. If a free user exceeds 5GB/month:
   - deactivate peer
   - remove `tc` peer classes and filters
   - reject future connect until the next monthly period

**Important Rule**

Set CAKE bandwidth to the bottleneck it actually owns:

| CAKE Location | CAKE Bandwidth |
|---|---:|
| Tier-level CAKE only | tier pool speed, such as `200mbit` or `800mbit` |
| Per-user CAKE leaf | user cap, such as `10mbit` or `100mbit` |
| Root CAKE on whole server | measured server capacity, such as `950mbit` |

For this design, use per-user CAKE leaves and set each one to that user's cap. Do not set every CAKE qdisc to `1gbit`.

**Final Recommendation**

Use:

```text
HTB root capacity = measured server capacity
HTB tier classes = ratio of capacity, free 20%, paid 80%
HTB peer classes = per-user cap, tier-dependent and server-policy-dependent
CAKE leaf per peer = same as that peer cap
WireGuard byte counter job = monthly quota enforcement for free users
```

This scales cleanly from 1Gbps to 10Gbps, lets you override per-server user limits, and avoids CAKE behaving like almost-unlimited bandwidth under smaller HTB classes.
