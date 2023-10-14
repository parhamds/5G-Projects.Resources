<!--
SPDX-License-Identifier: Apache-2.0
-->

# Resources
This project uses SD-Core as a base to run the Virtual UPF

### Setup SD-Core

```
sudo sysctl fs.inotify.max_user_instances=2147483647
sudo sysctl fs.inotify.max_user_watches=2147483647
git clone "https://gerrit.opencord.org/aether-in-a-box"
```
Change aether-in-a-box/resources/router.yaml to this config
```
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: router-net
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "${DATA_IFACE}",
    "ipam": {
        "type": "static"
    }
  }'
---
apiVersion: v1
kind: Pod
metadata:
  name: router
  labels:
    app: router
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
            { "name": "router-net", "interface": "exitlb-gw", "ips": ["192.168.254.1/24"] },
            { "name": "router-net", "interface": "ran-gw", "ips": ["192.168.251.1/24"] },
            { "name": "router-net", "interface": "enterlb-gw", "ips": ["192.168.255.1/24"] }
    ]'
spec:
  containers:
  - name: router
    command: ["/bin/bash", "-c"]
    args:
      - >
        sysctl -w net.ipv4.ip_forward=1;
        iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE;
        ip route add 172.250.0.0/16 via 192.168.254.2;
        ip route add 192.168.252.0/24 via 192.168.255.2;
        trap : TERM INT; sleep infinity & wait
    image: weibeld/ubuntu-networking
    securityContext:
      privileged: true
      capabilities:
        add:
          - NET_ADMIN
```
In aether-in-a-box/Makefile change the HELM_GLOBAL_ARGS in line 80 to prevent timeout
```
HELM_GLOBAL_ARGS ?= --timeout=45m
```

Then Run
```
cd aether-in-a-box
CHARTS=release-2.0 make roc-5g-models
CHARTS=release-2.0 make router-pod
kubectl create namespace omec
git clone https://github.com/parhamds/Resources.git
kubectl apply -n omec -f resources
```
Change the config of interfaces, PFCP-LB, Neworks as it fits your needs.

Then Run:
```
kubectl apply -n omec -f Resources
```
To Start the test using gNBSim:
```
kubectl -n omec exec gnbsim-0 -- ./gnbsim 2>&1 | tee /tmp/gnbsim.out; \
grep -q "Simulation Result: PASS\|Profile Status: PASS" /tmp/gnbsim.out
```
