# Cilium - Geneve Encapsulation + IPsec Encryption Performance Test

## Test Setup

| Parameter | Value |
| --- | --- |
| AKS Kubernetes version | 1.35.1 |
| Node SKUs | Standard_D4das_v5 (4 vCPU, 16 GB RAM), Standard_D4ds_v5 (4 vCPU, 16 GB RAM) |
| Networking mode | BYOCNI |
| Cilium version | 1.19.3 |
| Routing mode | Tunnel |
| Encapsulation protocol | Geneve |
| Encryption | IPsec |
| Test tools | iperf3, netperf |
| Test duration | 30 seconds |

### BYOCNI - Cilium specific configuration parameters

```YAML
aksbyocni:
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
kubeProxyReplacement: true
routingMode: tunnel
socketLB:
  hostNamespaceOnly: true
tunnelProtocol: geneve
```

## Tests

### TCP and UDP throughput tests using iperf3

```bash
# TCP throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -t 30 -P 4

[SUM]   0.00-30.00  sec  8.49 GBytes  2.43 Gbits/sec  32514             sender
[SUM]   0.00-30.04  sec  8.48 GBytes  2.43 Gbits/sec                  receiver
```

```bash
# UDP throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -u -b 10G -t 30

[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-30.00  sec  2.71 GBytes   776 Mbits/sec  0.000 ms  0/2201514 (0%)  sender
[  5]   0.00-30.04  sec  2.70 GBytes   772 Mbits/sec  0.015 ms  8200/2201511 (0.37%)  receiver
```

```bash
# Bidirectional throughput:
kubectl exec -n netperf -it netperf-client -- iperf3 -c iperf3-server -t 30 --bidir

[ ID][Role] Interval           Transfer     Bitrate         Retr
[  5][TX-C]   0.00-30.00  sec  1.88 GBytes   538 Mbits/sec  682             sender
[  5][TX-C]   0.00-30.04  sec  1.87 GBytes   536 Mbits/sec                  receiver
[  7][RX-C]   0.00-30.00  sec  8.07 GBytes  2.31 Gbits/sec  1104             sender
[  7][RX-C]   0.00-30.04  sec  8.07 GBytes  2.31 Gbits/sec                  receiver
```

### TCP and UDP latency tests using netperf

```bash
# TCP latency:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t TCP_RR -l 30 -- -O min_latency,mean_latency,max_latency,p99_latency,stddev_latency,transaction_rate

Minimum      Mean         Maximum      99th         Stddev       Transaction
Latency      Latency      Latency      Percentile   Latency      Rate
Microseconds Microseconds Microseconds Latency      Microseconds Tran/s
                                       Microseconds
164          197.71       6705         277          71.93        5054.313
```

```bash
# UDP latency:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t UDP_RR -l 30 -- -O min_latency,mean_latency,max_latency,p99_latency,stddev_latency,transaction_rate

Minimum      Mean         Maximum      99th         Stddev       Transaction
Latency      Latency      Latency      Percentile   Latency      Rate
Microseconds Microseconds Microseconds Latency      Microseconds Tran/s
                                       Microseconds
161          197.23       7004         292          68.07        5066.559
```

```bash
# TCP stream:
kubectl exec -n netperf -it netperf-client -- netperf -H netperf-server -t TCP_STREAM -l 30

Recv   Send    Send
Socket Socket  Message  Elapsed
Size   Size    Size     Time     Throughput
bytes  bytes   bytes    secs.    10^6bits/sec

131072  16384  16384    30.00    2367.54
```
