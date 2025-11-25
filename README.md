# Kubernetes VoIP Stack - Deployment Guide

Complete guide for deploying Kamailio + FreeSWITCH + RTPengine on Kubernetes.

---

## 🏗️ Architecture Overview

```
                 ┌─── Internet (SIP Clients) ───┐
                 │                               │
                 ▼                               ▼
        ┌──────────────────┐         ┌──────────────────┐
        │  AWS ELB/NLB     │         │  Node Public IP  │
        │   (SIP 5060)     │         │  (RTP 10000+)    │
        └────────┬─────────┘         └────────┬─────────┘
                 │                            │
                 ▼                            ▼
┌────────────────────────────────┐  ┌─────────────────────┐
│     Kamailio (Any Node)        │  │  RTPengine (Pinned) │
│       Pod Network              │  │   hostNetwork       │
│       :5060 UDP                │──│   Node IP:22222     │
│                                │  │   :10000-20000 UDP  │
└────────────┬───────────────────┘  └─────────────────────┘
             │                              ▲
             │ Connects via Node IP ────────┘
             │
             ▼
   ┌──────────────┐         ┌─────────────────┐
   │ FreeSWITCH   │         │   PostgreSQL    │
   │ (Any Node)   │         │   (Any Node)    │
   │  Pod Network │         │   Pod Network   │
   └──────────────┘         └─────────────────┘
```

---

## 📦 Components

### **1. Kamailio (SIP Proxy)**
- **Role**: SIP registration, authentication, call routing
- **Network**: Pod network (ClusterIP) - can run on **any node**
- **Ports**: 
  - 5060/UDP - SIP signaling
  - Exposed via LoadBalancer
- **Database**: PostgreSQL (kamailio DB)
- **Connects to RTPengine**: Via node IP (e.g., `192.168.32.28:22222`)

### **2. RTPengine (Media Proxy)**
- **Role**: RTP/RTCP media relay and transcoding
- **Network**: `hostNetwork: true` (REQUIRED) - **pinned to specific node**
- **Ports**:
  - 22222/UDP - Control interface (listens on node IP for Kamailio)
  - 10000-20000/UDP - RTP media ports (listens on node IP)
- **Public IP Discovery**: Automatically discovers node's public IP using `curl ifconfig.me`
- **Why hostNetwork?**: 
  - Needs to bind to node's network interface for RTP traffic
  - Must advertise public IP in SDP for external clients
  - Kubernetes doesn't support port ranges in services

### **3. FreeSWITCH (Media Gateway)**
- **Role**: Media server, handles calls forwarded by Kamailio
- **Network**: Pod network (ClusterIP) - can run on **any node**
- **Ports**: 5060/UDP, 5061/TCP, 8021/TCP
- **Database**: PostgreSQL (freeswitch DB)

### **4. PostgreSQL Databases**
- **postgres-kamailio**: Stores SIP users, location, dispatcher
- **postgresql-freeswitch**: FreeSWITCH configuration database
- **Network**: Standard pod network (StatefulSet)

---

## ⚠️ Critical Requirements

### **🔴 Only RTPengine Needs hostNetwork**

**RTPengine** MUST use `hostNetwork: true`. Here's why:

#### Why RTPengine Needs hostNetwork:
```
✅ RTP Port Ranges (10000-20000)
   - Kubernetes Services don't support port ranges
   - Must bind directly to node's network interface
   - External clients send RTP to node's public IP

✅ Public IP Advertisement
   - Clients on internet need to reach RTP ports
   - RTPengine advertises node's public IP in SDP
   - AWS/Cloud routes traffic: Public IP → Private IP → RTPengine

✅ Direct Network Access
   - No NAT layers between clients and RTPengine
   - Low latency, high performance RTP processing
```

#### Kamailio Can Use Pod Network:
```
✅ Kamailio runs in pod network (ClusterIP)
   - Connects to RTPengine via node IP (e.g., 192.168.32.28:22222)
   - Can scale across multiple nodes
   - More flexible deployment
```

### **🟢 Flexible Scheduling**

**Current Architecture:**
- ✅ **Kamailio**: Can run on **any node** (uses pod network)
- ✅ **FreeSWITCH**: Can run on **any node** (uses pod network)
- ⚠️ **RTPengine**: Pinned to **specific node** (uses hostNetwork)

**RTPengine Node Selection:**
```yaml
nodeSelector:
  kubernetes.io/hostname: ip-192-168-32-28.ap-south-1.compute.internal
```

**Why RTPengine is Pinned:**
- Must bind to specific node's network interface
- Advertises that node's public IP for RTP traffic
- **Only ONE RTPengine per node** due to hostNetwork port conflicts
- Update `RTPENGINE_SOCK` in kamailio-configmap.yaml if you move RTPengine

**Node Taints (Recommended):**
```bash
# Taint the RTPengine node to prevent other pods from scheduling there
kubectl taint nodes ip-192-168-32-28.ap-south-1.compute.internal \
  rtpengine=true:NoSchedule

# RTPengine deployment tolerates this taint
```

This ensures:
- Other pods don't consume resources on the RTPengine node
- RTPengine has dedicated resources for media processing
- No hostNetwork port conflicts with other applications

**Kamailio High Availability:**
```yaml
# Kamailio uses podAntiAffinity to spread replicas
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector: ...
        topologyKey: kubernetes.io/hostname
# Each Kamailio replica runs on different node
```

---

## 🚀 Deployment

### **Prerequisites**

1. **Kubernetes cluster** (tested on EKS 1.28+)
2. **AWS Security Group** must allow:
   - UDP 5060 (SIP signaling)
   - UDP 10000-20000 (RTP media) from `0.0.0.0/0`
3. **Update RTPengine node** in `k8s/rtpengine.yaml`:
   ```yaml
   nodeSelector:
     kubernetes.io/hostname: <your-node-name>
   ```
4. **Update RTPengine address** in `k8s/kamailio-configmap.yaml`:
   ```yaml
   #!define RTPENGINE_SOCK "udp:<node-private-ip>:22222"
   ```

### **Deploy All Components**

```bash
# 1. Create namespace
kubectl apply -f k8s/namespace.yaml

# 2. Deploy databases
kubectl apply -f k8s/postgres-kamailio.yaml
kubectl apply -f k8s/postgres-freeswitch.yaml

# 3. Deploy FreeSWITCH
kubectl apply -f k8s/freeswitch.yaml

# 4. Deploy RTPengine (must be before Kamailio due to affinity)
kubectl apply -f k8s/rtpengine.yaml

# 5. Deploy Kamailio config and service
kubectl apply -f k8s/kamailio-configmap.yaml
kubectl apply -f k8s/kamailio.yaml

# 6. Verify deployment
kubectl get pods -n sip -o wide
```

### **Verify RTPengine Connection**

```bash
# Check if Kamailio detected RTPengine
kubectl logs -n sip -l app=kamailio --tail=20 | grep rtpengine

# Expected output:
# INFO: rtpengine instance <udp:127.0.0.1:22222> found, support for it enabled
```

---

## 🔧 Configuration

### **RTPengine Node Selection**

RTPengine is pinned to a specific node using hostname selector:

```yaml
nodeSelector:
  kubernetes.io/hostname: ip-192-168-32-28.ap-south-1.compute.internal
```

**Taint the RTPengine node (Recommended):**
```bash
# Prevent other pods from scheduling on RTPengine node
kubectl taint nodes ip-192-168-32-28.ap-south-1.compute.internal \
  rtpengine=true:NoSchedule
```

**Move RTPengine to different node:**
```bash
# 1. Remove taint from old node
kubectl taint nodes old-node rtpengine-

# 2. Taint new node
kubectl taint nodes new-node rtpengine=true:NoSchedule

# 3. Update rtpengine.yaml with new hostname
# 4. Update kamailio-configmap.yaml with new node's private IP
# 5. Apply changes
```

### **Automatic Public IP Discovery**

RTPengine automatically discovers the node's public IP using `ifconfig.me`:

```yaml
env:
  - name: PRIVATE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP

command:
  - /bin/sh
args:
  - -c
  - |
    # Auto-discover public IP
    EXTERNAL_IP=$(curl -s --max-time 5 ifconfig.me || echo "${PRIVATE_IP}")
    
    # Bind to private IP, advertise public IP in SDP
    /usr/sbin/rtpengine \
      --interface=${PRIVATE_IP}!${EXTERNAL_IP} \
      --listen-ng=${PRIVATE_IP}:22222 ...
```

**How it works:**
- `PRIVATE_IP`: Node's private IP (e.g., `192.168.32.28`) - RTPengine binds to this
- `EXTERNAL_IP`: Node's public IP (e.g., `35.154.207.150`) - Advertised in SDP to clients
- Clients send RTP to public IP → AWS routes to private IP → RTPengine receives packets

---

## 🧪 Testing

### **1. Register SIP Clients**

Configure your SIP softphone (Zoiper, Linphone, etc.):

```
Server: <node-ip>:30060
Username: 1001 (or register via location table)
Password: 1234
Transport: UDP

Server: <node-ip>:30060
Username: 1002 (or register via location table)
Password: 1234
Transport: UDP
```

### **2. Make a Call**

Call between extensions (e.g., 1001 → 1002)

### **3. Verify RTP Processing**

```bash
# Check if RTPengine is processing media
kubectl logs -n sip -l app=rtpengine --tail=50 | grep -E "(offer|answer|packet)"

# Expected output:
# INFO: Received command 'offer' from 127.0.0.1:xxxxx
# INFO: Received command 'answer' from 127.0.0.1:xxxxx
# INFO: RTP packet received
```

### **4. Check Call Flow**

```bash
# Kamailio logs
kubectl logs -n sip -l app=kamailio --tail=50 | grep -E "(INVITE|200|Extension)"

# RTPengine stats after call
kubectl logs -n sip -l app=rtpengine --tail=50 | grep "Final packet stats"
```

---

## 🐛 Troubleshooting

### **No Audio**

**Symptom**: Call connects but no audio

**Diagnosis:**
```bash
# 1. Check if RTPengine is receiving commands
kubectl logs -n sip -l app=rtpengine --tail=50 | grep offer
# Should see: "Received command 'offer'"

# 2. Check if answer is being processed
kubectl logs -n sip -l app=rtpengine --tail=50 | grep answer
# Should see: "Received command 'answer'"

# 3. Check RTP packet flow
kubectl logs -n sip -l app=rtpengine --tail=100 | grep "packet stats"
# Should show packets in both directions, NOT "(null):0"
```

**Common Causes:**
1. **RTPengine not connected**: Check `kubectl logs -n sip -l app=kamailio | grep rtpengine`
2. **AWS Security Group blocking UDP 10000-20000**: Check inbound rules
3. **Public IP not advertised**: Check RTPengine logs for "Public IP: X.X.X.X"
4. **Wrong RTPengine address**: Check `RTPENGINE_SOCK` in kamailio-configmap.yaml

### **Kamailio Can't Reach RTPengine**

**Error**: `timeout waiting reply` or `Connection refused:111`

**Fix:**
```bash
# 1. Check RTPengine address in Kamailio config
kubectl get configmap -n sip kamailio-config -o yaml | grep RTPENGINE_SOCK
# Should match RTPengine node's private IP, e.g.: udp:192.168.32.28:22222

# 2. Verify RTPengine is listening on correct IP
kubectl logs -n sip -l app=rtpengine | grep "Listen NG"
# Should show: Listen NG: 192.168.32.28:22222

# 3. Test connectivity from Kamailio pod
kubectl exec -n sip -l app=kamailio -- nc -uzv 192.168.32.28 22222

# 4. Restart Kamailio if config changed
kubectl rollout restart deployment -n sip kamailio
```

### **Need to Move RTPengine to Different Node**

**Scenario**: Want to use a different node for RTPengine

**Steps:**
```bash
# 1. Get new node's details
kubectl get nodes -o wide
# Note: hostname and private IP

# 2. Update rtpengine.yaml
#    Change nodeSelector to new hostname

# 3. Update kamailio-configmap.yaml
#    Change RTPENGINE_SOCK to new node's private IP

# 4. Apply changes
kubectl apply -f k8s/rtpengine.yaml
kubectl apply -f k8s/kamailio-configmap.yaml
kubectl rollout restart deployment -n sip kamailio

# 5. Verify
kubectl logs -n sip -l app=rtpengine | grep -E "(Private|Public) IP"
```

---

## 📊 Monitoring

### **Check Pod Status**
```bash
kubectl get pods -n sip -o wide
```

### **Check Resource Usage**
```bash
kubectl top pods -n sip
```

### **View Logs**
```bash
# Real-time Kamailio logs
kubectl logs -n sip -l app=kamailio -f

# Real-time RTPengine logs
kubectl logs -n sip -l app=rtpengine -f

# Recent call logs
kubectl logs -n sip -l app=kamailio --tail=100 | grep INVITE
```

### **Check RTPengine Health**
```bash
# See active calls
kubectl logs -n sip -l app=rtpengine | grep "Creating new call"

# Check for errors
kubectl logs -n sip -l app=rtpengine | grep -i error
```

---

## 📈 Scaling

### **Horizontal Scaling**

**Can Scale Horizontally:**
- ✅ **Kamailio**: Multiple replicas across nodes (uses podAntiAffinity)
- ✅ **FreeSWITCH**: Multiple replicas with dispatcher
- ✅ **PostgreSQL**: Read replicas for high availability

**Limited Scaling:**
- ⚠️ **RTPengine**: **ONE per node** (hostNetwork limitation)
  - Cannot run multiple RTPengine instances on same node
  - Port conflicts: All bind to same ports (22222, 10000-20000)
  - To scale RTP capacity: Deploy on multiple nodes

**Multi-RTPengine Setup (Advanced):**

To scale RTP capacity across multiple nodes:

```bash
# 1. Taint additional nodes for RTPengine
kubectl taint nodes node-2 rtpengine=true:NoSchedule
kubectl taint nodes node-3 rtpengine=true:NoSchedule

# 2. Create separate deployments (rtpengine-2, rtpengine-3)
#    Each with unique nodeSelector pointing to different nodes

# 3. Configure Kamailio dispatcher module to load balance across RTPengines

# Kamailio scales horizontally
kubectl scale deployment kamailio -n sip --replicas=3
```

---

## 🗂️ File Structure

```
k8s/
├── namespace.yaml                 # sip namespace
├── postgres-kamailio.yaml         # Kamailio database
├── postgres-freeswitch.yaml       # FreeSWITCH database
├── kamailio-configmap.yaml        # Kamailio SIP configuration
├── kamailio.yaml                  # Kamailio deployment + service
├── rtpengine.yaml                 # RTPengine deployment
└── freeswitch.yaml                # FreeSWITCH deployment + service
```

---

## 📝 Notes

1. **hostNetwork ONLY for RTPengine** - Kamailio and FreeSWITCH use pod network
2. **RTPengine pinned to specific node** - uses hostname in nodeSelector
3. **ONE RTPengine per node** - hostNetwork causes port conflicts if multiple on same node
4. **Taint RTPengine node** - prevents other pods from consuming resources
5. **Automatic public IP discovery** - uses `curl ifconfig.me` on pod startup
6. **Kamailio connects via node IP** - not localhost (e.g., `192.168.32.28:22222`)
7. **AWS Security Group** must allow UDP 10000-20000 from `0.0.0.0/0`
8. **Kamailio can scale** - podAntiAffinity spreads replicas across nodes
9. **Update two files when moving RTPengine**: rtpengine.yaml (nodeSelector) and kamailio-configmap.yaml (RTPENGINE_SOCK)

---

## 🤝 Support

For issues:
1. Check troubleshooting section above
2. Verify RTPengine logs show both offer and answer commands
3. Ensure both Kamailio and RTPengine are on same node with hostNetwork
4. Verify firewall allows UDP 10000-20000 to node IP

---

## 📄 License

MIT License - Feel free to use and modify for your VoIP deployments.

---

## 👨‍💻 Author

**Vishal Kapadi**  
DevOps Engineer

- 🐙 **GitHub:** [github.com/vishalk17](https://github.com/vishalk17)
- 🎥 **YouTube:** [youtube.com/@vishalk17](https://www.youtube.com/@vishalk17)
- 💼 **LinkedIn:** [linkedin.com/in/vishal-kapadi](https://www.linkedin.com/in/vishal-kapadi/)

Subscribe to my YouTube channel for DevOps deep-dives!

---

**© 2025 Vishal Kapadi. All rights reserved.**
