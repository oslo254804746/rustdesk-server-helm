[中文文档](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README_zh.MD)


[English](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README.MD)

## Introduction
This is a chart for rustdesk-server

## Configuration Options
### Port Configuration
According to the official documentation of `rustdesk`, there are five ports in total, `21118-21119` are web ports, which are not required to be opened, 

and `21116` needs to include TCP/UDP. The default `PortType` is `ClusterIP`, which can be modified to `NodePort` or Proxy by yourself through  like `traefik crd`
```yaml
portConfigs:
  hbbr:
    type: ClusterIP
    replayPort:
      port: 21117
      targetPort: 21117
    clientPort:
      enable: false
      port: 21119
      targetPort: 21119
  hbbs:
      type: ClusterIP
      natPort:
        port: 21115
        protocol: TCP
        targetPort: 21115
      registryPort:
        port: 21116
        targetPort: 21116
      heartbeatPort:
        port: 21116
        targetPort: 21116
      webPort:
        port: 21118
        targetPort: 21118
        enable: false
        protocol: TCP
 ```

### Volume Configuration
The default is EmptyDir, which is shared by two containers, storing pubKey and SQLite data
```yaml
volumes:
  - name: rustdesk-data
    emptyDir: {}
```
You can modify it yourself, such as changing it to hostPath
```yaml
volumes:
  - name: rustdesk-data
    hostPath:
      path: /mnt/rustdesk/
```


## Network Configuration

### Host Network vs LoadBalancer/NodePort

#### Option 1: Host Network (Simple, Direct Access)
When enabled, pods use the host's network namespace directly:

```yaml
hostNetwork: true
portConfigs:
  hbbs:
    type: ClusterIP  # or NodePort
  hbbr:
    type: ClusterIP
```

**Pros:** Simple setup, direct host network access  
**Cons:** 
- Limits one pod per node (port conflicts)
- Reduces network isolation
- Requires available ports on host

#### Option 2: LoadBalancer/NodePort (Recommended for Production)
Use Kubernetes services with proper external access:

```yaml
hostNetwork: false  # IMPORTANT: Must be false!

rustdesk:
  relayServer: "YOUR.LOADBALANCER.IP"  # REQUIRED: Your external IP

service:
  externalTrafficPolicy: Local  # Preserves client source IP
  hbbs:
    loadBalancerIP: "YOUR.LOADBALANCER.IP"
  hbbr:
    loadBalancerIP: "YOUR.LOADBALANCER.IP"

portConfigs:
  hbbs:
    type: LoadBalancer
  hbbr:
    type: LoadBalancer
```

**Why `rustdesk.relayServer` is Required:**  
When using LoadBalancer, RustDesk needs to know its external IP to advertise to clients for peer-to-peer connections. Without this, clients will try to connect to the internal pod IP and timeout.

### Service Annotations and Labels

Add custom annotations and labels to services for cloud providers, monitoring, or service mesh integration:

```yaml
service:
  # Global annotations applied to both hbbs and hbbr
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "21118"
  
  # Global labels applied to both services
  labels:
    team: infrastructure
    environment: production
  
  # Service-specific configurations
  hbbs:
    loadBalancerIP: "192.168.1.100"
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    labels:
      component: signal-server
  
  hbbr:
    loadBalancerIP: "192.168.1.100"
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    labels:
      component: relay-server
```

**Note:** Service-specific annotations/labels override global ones for the same key.

### MetalLB Configuration

For bare-metal Kubernetes clusters using MetalLB:

```yaml
hostNetwork: false

rustdesk:
  relayServer: "192.168.1.100"  # Your MetalLB IP

service:
  externalTrafficPolicy: Local
  annotations:
    metallb.universe.tf/allow-shared-ip: "rustdesk"  # Required for shared IP
  hbbs:
    loadBalancerIP: "192.168.1.100"
  hbbr:
    loadBalancerIP: "192.168.1.100"  # Same IP, different ports

portConfigs:
  hbbs:
    type: LoadBalancer
  hbbr:
    type: LoadBalancer
```

### Cloud Provider Examples

#### AWS Network Load Balancer
```yaml
service:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
  hbbs:
    loadBalancerIP: ""  # Auto-assign
  hbbr:
    loadBalancerIP: ""
```

#### GCP Internal Load Balancer
```yaml
service:
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
    networking.gke.io/load-balancer-type: "Internal"
```

#### Azure Internal Load Balancer
```yaml
service:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "subnet-name"
```

## Configuration Parameters

### Core Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of pod replicas | `1` |
| `image.repository` | RustDesk server image | `rustdesk/rustdesk-server` |
| `image.tag` | Image tag | `1.1.9` |
| `hostNetwork` | Use host network namespace | `false` |

### RustDesk Configuration

| Parameter | Description | Default | Required |
|-----------|-------------|---------|----------|
| `rustdesk.relayServer` | External IP for client connections | `""` | Yes, when using LoadBalancer |

### Service Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.externalTrafficPolicy` | Traffic policy (Local/Cluster) | `Local` |
| `service.annotations` | Global service annotations | `{}` |
| `service.labels` | Global service labels | `{}` |
| `service.hbbs.loadBalancerIP` | LoadBalancer IP for hbbs | `""` |
| `service.hbbs.annotations` | hbbs-specific annotations | `{}` |
| `service.hbbs.labels` | hbbs-specific labels | `{}` |
| `service.hbbr.loadBalancerIP` | LoadBalancer IP for hbbr | `""` |
| `service.hbbr.annotations` | hbbr-specific annotations | `{}` |
| `service.hbbr.labels` | hbbr-specific labels | `{}` |

### Port Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `portConfigs.hbbs.type` | Service type for hbbs | `ClusterIP` |
| `portConfigs.hbbr.type` | Service type for hbbr | `ClusterIP` |

See values.yaml for complete port configuration options.

## Installation

### Basic Installation (Host Network)
```shell
cd rustdesk-server-helm
helm install rustdesk . --set hostNetwork=true
```

### LoadBalancer Installation
```shell
helm install rustdesk . \
  --set hostNetwork=false \
  --set rustdesk.relayServer="YOUR.LOADBALANCER.IP" \
  --set portConfigs.hbbs.type=LoadBalancer \
  --set portConfigs.hbbr.type=LoadBalancer \
  --set service.hbbs.loadBalancerIP="YOUR.LOADBALANCER.IP" \
  --set service.hbbr.loadBalancerIP="YOUR.LOADBALANCER.IP"
```

### MetalLB Installation
```shell
helm install rustdesk . \
  --set hostNetwork=false \
  --set rustdesk.relayServer="192.168.1.100" \
  --set portConfigs.hbbs.type=LoadBalancer \
  --set portConfigs.hbbr.type=LoadBalancer \
  --set service.hbbs.loadBalancerIP="192.168.1.100" \
  --set service.hbbr.loadBalancerIP="192.168.1.100" \
  --set service.annotations."metallb\.universe\.tf/allow-shared-ip"="rustdesk"
```

## Troubleshooting

### Connection Timeout with LoadBalancer

**Symptom:** Works with hostNetwork but times out with LoadBalancer

**Cause:** RustDesk advertises internal pod IP instead of LoadBalancer IP to clients

**Solution:** Set `rustdesk.relayServer` to your LoadBalancer IP:
```yaml
rustdesk:
  relayServer: "YOUR.LOADBALANCER.IP"
```

### MetalLB Shared IP Not Working

Ensure you have the annotation:
```yaml
service:
  annotations:
    metallb.universe.tf/allow-shared-ip: "rustdesk"
```

### UDP Traffic Not Working

- Verify your LoadBalancer supports UDP (AWS ELB doesn't, use NLB)
- Check firewall rules allow UDP on port 21116
- Verify `externalTrafficPolicy: Local` is set

## Technical Notes

- **Label/Annotation Merging:** Service-specific values override global values using `mergeOverwrite`
- **External Traffic Policy:** `Local` preserves client source IP but requires healthy local endpoints
- **Shared IP:** Both services can share one IP as they use different ports (21115-21117)
