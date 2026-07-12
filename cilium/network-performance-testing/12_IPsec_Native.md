# Cilium - Native Routing + IPsec Encryption Performance Test

## Test Setup

| Parameter | Value |
| --- | --- |
| AKS Kubernetes version | 1.35.1 |
| Node SKUs | Standard_D4das_v5 (4 vCPU, 16 GB RAM), Standard_D4ds_v5 (4 vCPU, 16 GB RAM) |
| Networking mode | BYOCNI |
| Cilium version | 1.19.3 |
| Routing mode | Native |
| Encapsulation protocol | None |
| Encryption | IPsec |
| Test tools | iperf3, netperf |
| Test duration | 30 seconds |

> [!NOTE]
> Additional Azure resource setup required: [An experiment – Enable Cilium native routing on Azure Kubernetes Service BYOCNI – Part 3](https://www.danielstechblog.io/an-experiment-enable-cilium-native-routing-on-azure-kubernetes-service-byocni-part-3/)
>
> - BGP Route Reflector VM using Ubuntu + FRR
> - Azure Route Server

### BYOCNI - Cilium specific configuration parameters

```YAML
aksbyocni:
  enabled: false
autoDirectNodeRoutes: false
bgpControlPlane:
  enabled: true
bpf:
  hostLegacyRouting: false
  masquerade: true
ciliumEndPointSlice:
  enabled: true
cni:
  exclusive: true
disableEndPointCRD: false
dnsProxy:
  enableTransparentMode: true
encryption:
  enabled: true
  type: ipsec
ipam:
  operator:
    clusterPoolIPv4PodCIDRList:
      - 100.64.0.0/10
ipv4NativeRoutingCIDR: 100.64.0.0/10
kubeProxyReplacement: true
routingMode: native
socketLB:
  hostNamespaceOnly: true
```

### Cilium configuration check

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium status | grep -E "Masquerading|Routing|Encryption"
KubeProxyReplacement:    True   [eth0   10.10.0.10 fe80::7eed:8dff:fe46:c106 (Direct Routing)]
Routing:                 Network: Native   Host: BPF
Masquerading:            BPF   [eth0]   100.64.0.0/10  [IPv4: Enabled, IPv6: Disabled]
Encryption:              IPsec
```

## Tests

### TCP and UDP throughput tests using iperf3

```bash
# TCP throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -t 30 -P 4

[SUM]   0.00-30.00  sec  9.00 GBytes  2.58 Gbits/sec  4474             sender
[SUM]   0.00-30.04  sec  8.99 GBytes  2.57 Gbits/sec                  receiver
```

```bash
# UDP throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -u -b 10G -t 30

[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-30.00  sec  3.37 GBytes   964 Mbits/sec  0.000 ms  0/2635808 (0%)  sender
[  5]   0.00-30.04  sec  3.32 GBytes   949 Mbits/sec  0.010 ms  37995/2635794 (1.4%)  receiver
```

```bash
# Bidirectional throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -t 30 --bidir

[ ID][Role] Interval           Transfer     Bitrate         Retr
[  5][TX-C]   0.00-30.00  sec  6.84 GBytes  1.96 Gbits/sec  785             sender
[  5][TX-C]   0.00-30.05  sec  6.83 GBytes  1.95 Gbits/sec                  receiver
[  7][RX-C]   0.00-30.00  sec  1.85 GBytes   529 Mbits/sec  1032             sender
[  7][RX-C]   0.00-30.05  sec  1.85 GBytes   528 Mbits/sec                  receiver
```

### TCP and UDP latency tests using netperf

```bash
# TCP latency:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t TCP_RR -l 30 -- -O min_latency,mean_latency,max_latency,p99_latency,stddev_latency,transaction_rate

Minimum      Mean         Maximum      99th         Stddev       Transaction
Latency      Latency      Latency      Percentile   Latency      Rate
Microseconds Microseconds Microseconds Latency      Microseconds Tran/s
                                       Microseconds
151          182.72       5083         253          95.34        5468.312
```

```bash
# UDP latency:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t UDP_RR -l 30 -- -O min_latency,mean_latency,max_latency,p99_latency,stddev_latency,transaction_rate

Minimum      Mean         Maximum      99th         Stddev       Transaction
Latency      Latency      Latency      Percentile   Latency      Rate
Microseconds Microseconds Microseconds Latency      Microseconds Tran/s
                                       Microseconds
148          180.38       5865         248          92.47        5539.226
```

```bash
# TCP stream:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t TCP_STREAM -l 30

Recv   Send    Send
Socket Socket  Message  Elapsed
Size   Size    Size     Time     Throughput
bytes  bytes   bytes    secs.    10^6bits/sec

131072  16384  16384    30.01    2552.48
```
