# SONiC dual-LAN k3s cluster with nginx anti-affinity

A 3-node k3s cluster spread across two physically separated LANs, routed by a SONiC switch. nginx runs with `requiredDuringSchedulingIgnoredDuringExecution` pod anti-affinity so each replica lands on a different node — and therefore spans both LANs.

Built on [Antrieb](https://antrieb.sh/dash). Cluster provisioned in **946 ms**; total wall time end-to-end including k3s install, nginx rollout, and verification was a couple of minutes.

## Topology

```
                       ┌──────────────────────┐
                       │      node1 (SONiC)   │
                       │                      │
              eth0 ┌───┤ 10.10.1.254/24       │──┐
                   │   │ 10.10.2.254/24       │  │eth1
                   │   │ ip_forward=1         │  │
                   │   └──────────────────────┘  │
                   │                             │
        ┌──────────┴──────────┐    ┌─────────────┴───────┐
        │       lan-a         │    │       lan-b         │
        │  10.10.1.0/24 DHCP  │    │  10.10.2.0/24 DHCP  │
        └──────┬──────┬───────┘    └──────────┬──────────┘
               │      │                       │
        ┌──────┴──┐ ┌─┴───────┐         ┌─────┴────┐
        │ node2   │ │ node3   │         │ node4    │
        │ 10.10.  │ │ 10.10.  │         │ 10.10.   │
        │ 1.11    │ │ 1.12    │         │ 2.11     │
        │ k3s     │ │ k3s     │         │ k3s      │
        │ server  │ │ agent   │         │ agent    │
        └─────────┘ └─────────┘         └──────────┘
```

| Node    | Image          | LAN    | IP            | Role                       |
|---------|----------------|--------|---------------|----------------------------|
| node1   | `sonic`        | a + b  | .254 on each  | L3 router between LANs     |
| node2   | `ubuntu24.04`  | lan-a  | 10.10.1.11    | k3s server + worker        |
| node3   | `ubuntu24.04`  | lan-a  | 10.10.1.12    | k3s agent                  |
| node4   | `ubuntu24.04`  | lan-b  | 10.10.2.11    | k3s agent                  |

- The two LANs are independent L2 segments. Their only connection is through SONiC's two NICs.
- Ubuntu nodes use DHCP. SONiC takes static `.254` on each LAN (the platform reserves `.1`).
- Both LANs have `egress: true` so apt + the k3s installer can reach the internet.

## Provision (Antrieb)

```jsonc
provision({
  cluster: [
    { image: "sonic" },
    { image: "ubuntu24.04", count: 3 }
  ],
  networks: [
    { name: "lan-a", cidr: "10.10.1.0/24", egress: true, dhcp: true },
    { name: "lan-b", cidr: "10.10.2.0/24", egress: true, dhcp: true }
  ],
  nics: {
    node1: [{ net: "lan-a" }, { net: "lan-b" }],   // SONiC straddles both
    node2: [{ net: "lan-a" }],
    node3: [{ net: "lan-a" }],
    node4: [{ net: "lan-b" }]
  }
})
```

## Step 1 — Configure SONiC as L3 router

SONiC-VS's dataplane is the Linux kernel (FIB + bridge), not SAI/ASIC. Treat `eth0..N` as real interfaces and configure with stock `ip`/`sysctl`.

Three platform gotchas to handle before assigning IPs:

1. `docker0` at `240.127.1.1/24` is up by default and steals source selection for ARP/ICMP — flush + down it.
2. `interfaces-config.service` is a oneshot that strips manually-assigned IPs during boot — stop it before configuring.
3. `net.ipv4.ip_forward=1` alone is not enough. Per-interface `net.ipv4.conf.<iface>.forwarding=1` is also required, otherwise the kernel silently black-holes forwarded packets.

```sh
# on node1 (SONiC)
systemctl stop interfaces-config.service
ip addr flush dev docker0
ip link set docker0 down

ip addr add 10.10.1.254/24 dev eth0
ip addr add 10.10.2.254/24 dev eth1

sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.eth0.forwarding=1
sysctl -w net.ipv4.conf.eth1.forwarding=1
```

## Step 2 — Cross-LAN static routes on Ubuntu nodes

DHCP gives each Ubuntu a default route via the platform gateway (`.1`), which only reaches the internet. The other LAN sits behind SONiC, so we add an explicit route.

```sh
# node2, node3 (lan-a)  -> reach lan-b through SONiC's lan-a IP
ip route add 10.10.2.0/24 via 10.10.1.254

# node4 (lan-b)         -> reach lan-a through SONiC's lan-b IP
ip route add 10.10.1.0/24 via 10.10.2.254
```

**Verification — cross-LAN ping, TTL=63 proves SONiC is forwarding L3:**

```
node2 $ ping -c 3 10.10.2.11
64 bytes from 10.10.2.11: icmp_seq=1 ttl=63 time=2.71 ms
64 bytes from 10.10.2.11: icmp_seq=2 ttl=63 time=1.32 ms
64 bytes from 10.10.2.11: icmp_seq=3 ttl=63 time=1.38 ms
3 packets transmitted, 3 received, 0% packet loss
```

`ttl=63` (not 64) is the proof — the TTL was decremented by one IP hop, which is SONiC. If SONiC were L2-bridging, TTL would stay at 64.

## Step 3 — Install k3s

Each node uses `--flannel-iface=enp1s0` so the Flannel VXLAN overlay rides on the LAN interface (which is routed through SONiC), not on docker0 or some auto-picked interface. `--node-ip` pins the address advertised to the API.

**Control plane (node2, lan-a):**

```sh
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
    --flannel-iface=enp1s0 \
    --node-ip=10.10.1.11 \
    --disable=traefik" sh -
```

Grab the join token:

```sh
cat /var/lib/rancher/k3s/server/node-token
```

**Agents (node3 on lan-a, node4 on lan-b):**

```sh
# node3
curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.10.1.11:6443 \
  K3S_TOKEN=<token> \
  INSTALL_K3S_EXEC="agent --flannel-iface=enp1s0 --node-ip=10.10.1.12" sh -

# node4
curl -sfL https://get.k3s.io | \
  K3S_URL=https://10.10.1.11:6443 \
  K3S_TOKEN=<token> \
  INSTALL_K3S_EXEC="agent --flannel-iface=enp1s0 --node-ip=10.10.2.11" sh -
```

node4 reaches the API on `10.10.1.11:6443` through SONiC. Same path Flannel uses for inter-node VXLAN.

**Verification:**

```
$ kubectl get nodes -o wide
NAME    STATUS   ROLES           VERSION         INTERNAL-IP
node2   Ready    control-plane   v1.35.4+k3s1    10.10.1.11
node3   Ready    <none>          v1.35.4+k3s1    10.10.1.12
node4   Ready    <none>          v1.35.4+k3s1    10.10.2.11
```

## Step 4 — nginx with required pod anti-affinity

The key bit is `requiredDuringSchedulingIgnoredDuringExecution` with `topologyKey: kubernetes.io/hostname`. "Required" means the scheduler will refuse to place two nginx pods on the same node; with 3 replicas and 3 nodes, you get exactly one pod per node.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels: {app: nginx}
spec:
  replicas: 3
  selector:
    matchLabels: {app: nginx}
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  template:
    metadata:
      labels: {app: nginx}
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels: {app: nginx}
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{containerPort: 80}]
          resources:
            requests: {cpu: "50m",  memory: "32Mi"}
            limits:   {cpu: "200m", memory: "128Mi"}
          readinessProbe:
            httpGet: {path: /, port: 80}
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /, port: 80}
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector: {app: nginx}
  ports: [{port: 80, targetPort: 80}]
```

```sh
kubectl apply -f nginx.yaml
```

## Step 5 — Tests

**Anti-affinity: one pod per node, spread across both LANs**

```
$ kubectl get pods -o wide -l app=nginx
NAME                     READY  STATUS   IP          NODE
nginx-7bcd86d597-pvvbg   1/1    Running  10.42.0.5   node2   # lan-a
nginx-7bcd86d597-6rfcp   1/1    Running  10.42.1.2   node3   # lan-a
nginx-7bcd86d597-c8j6l   1/1    Running  10.42.2.2   node4   # lan-b
```

3 distinct hostnames — anti-affinity is enforced.

**ClusterIP from node2 (lan-a):**

```
req 1 -> HTTP 200 (0.000374s)
req 2 -> HTTP 200 (0.002300s)
req 3 -> HTTP 200 (0.000322s)
req 4 -> HTTP 200 (0.001090s)
req 5 -> HTTP 200 (0.000411s)
```

**ClusterIP from node4 (lan-b) — proves kube-proxy + Flannel carry traffic cross-LAN through SONiC:**

```
$ for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code}\n" http://10.43.134.47/; done
200
200
200
200
200
200
200
200
200
200
```

**Pod-to-pod cross-LAN (node2 pod → node4 pod over Flannel VXLAN):**

```
$ kubectl exec nginx-...-pvvbg -- wget -qO- http://10.42.2.2/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

**Flannel overlay on node4 — riding the LAN interface, tunnel endpoints via SONiC:**

```
$ ip -br addr show flannel.1
flannel.1   UNKNOWN   10.42.2.0/32

$ ip route | grep 10.42
10.42.0.0/24 via 10.42.0.0 dev flannel.1 onlink   # node2 pod CIDR
10.42.1.0/24 via 10.42.1.0 dev flannel.1 onlink   # node3 pod CIDR
10.42.2.0/24 dev cni0 proto kernel scope link src 10.42.2.1  # local
```

## Notes / gotchas

- **`.1` is reserved by the platform's DHCP bridge.** Don't assign it to a router VM — duplicate IP causes ARP flip-flop. Use `.254`.
- **`docker0` on SONiC steals source selection.** Strip its IP before configuring the data plane, or outbound ARP/ICMP picks the wrong source.
- **`interfaces-config.service`** is a reconciler that removes manually-set IPs. Stop it before configuring, otherwise IPs vanish 10s later.
- **Global `ip_forward=1` is not enough on SONiC.** Per-interface `net.ipv4.conf.<iface>.forwarding=1` is also required, or forwarded packets are silently dropped.
- **`--flannel-iface=enp1s0`** is essential. Without it, Flannel may pick a non-LAN interface and the overlay won't traverse SONiC.
- The Ubuntu 24.04 image enumerates its single NIC as `enp1s0` (not `enp2s0`/`enp8s0`). Always `ip -br link` to confirm before scripting.

## Why this works at all

Three independent layers carry traffic between the two LANs, all over the same physical path (Ubuntu LAN NIC → SONiC eth0/eth1 → other LAN):

1. **Underlay routing:** static routes + SONiC's kernel FIB forward IP between `10.10.1.0/24` and `10.10.2.0/24`.
2. **k3s API:** node4's agent reaches `10.10.1.11:6443` over that underlay.
3. **Pod overlay:** Flannel VXLAN tunnels (`10.42.0.0/16`) are encapsulated in UDP on the LAN interfaces; the outer IP is the LAN IP, which is routed across LANs by SONiC.

Anti-affinity guarantees one pod per node, and because the nodes themselves span both LANs, the workload is automatically distributed across both physical segments.
