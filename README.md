# RKE2-Installation
Build a Kubernetes Cluster with RKE2 + Cilium CNI

# Install for RKE2 Master 1
curl -sfL https://get.rke2.io | sudo sh -

# Modify RKE2 Config
sudo mkdir -p /etc/rancher/rke2
sudo vim /etc/rancher/rke2/config.yaml

# Config 

write-kubeconfig-mode: "0644"
advertise-address: 192.168.100.50
tls-san:
  - 192.168.100.10
  - 192.168.100.11
  - 192.168.100.12
  - 192.168.100.21
  - 192.168.100.22
  - 192.168.100.23
  - 192.168.100.24
  - 192.168.100.25
cluster-cidr: 10.100.0.0/16
service-cidr: 10.110.0.0/16
cluster-dns: 10.110.0.10
cni: none
disable:
  - rke2-ingress-nginx
  - rke2-canal
disable-kube-proxy: true
token: "1234!@#$"
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
egress-selector-mode: disabled
protect-kernel-defaults: false  # Changed from true - can cause issues


# Enable for RKE2 service
sudo systemctl enable rke2-server
sudo systemctl start rke2-server

# Wait for RKE2 to be ready
sudo systemctl status rke2-server

# Set up kubectl
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc

# Cilium Install

cilium install \
    --namespace kube-system \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=dedicated \
    --set ingressController.default=true \
    --set gatewayAPI.enabled=true \
    --set kubeProxyReplacement=true \
    --set l2announcements.enabled=true \
    --set l2announcements.interfaces='{eth0}' \
    --set l2announcements.leaseDuration=15s \
    --set l2announcements.leaseRenewDeadline=5s \
    --set l2announcements.leaseRetryPeriod=2s \
    --set externalIPs.enabled=true \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set operator.prometheus.enabled=true \
    --set hubble.enabled=true \
    --set hubble.metrics.enableOpenMetrics=true \
    --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set ipam.mode=multi-pool \
    --set routingMode=native \
    --set autoDirectNodeRoutes=true \
    --set ipv4NativeRoutingCIDR=10.100.0.0/16 \
    --set endpointRoutes.enabled=true \
    --set bpf.masquerade=true \
    --set ipam.operator.autoCreateCiliumPodIPPools.default.ipv4.cidrs='{10.100.0.0/16}' \
    --set ipam.operator.autoCreateCiliumPodIPPools.default.ipv4.maskSize=24 \
    --set bpf.datapathMode=netkit \
    --set bpf.distributedLRU.enabled=true \
    --set ipv4.enabled=true \
    --set enableIPv4BIGTCP=true \
    --set bpfClockProbe=true \
    --set bandwidthManager.enabled=true \
    --set bandwidthManager.bbr=true


# Node Token Export

sudo cat /var/lib/rancher/rke2/server/node-token

# Join the 2nd and 3rd Master nodes

curl -sfL https://get.rke2.io | sudo sh -

sudo mkdir -p /etc/rancher/rke2
sudo vim /etc/rancher/rke2/config.yaml

# Config

write-kubeconfig-mode: "0644"
server: https://192.168.100.10:9345
token: "1234!@#$"
tls-san:
  - 192.168.100.10
  - 192.168.100.11
  - 192.168.100.12
  - 192.168.100.21
  - 192.168.100.22
  - 192.168.100.23
  - 192.168.100.24
  - 192.168.100.25
cluster-cidr: 10.100.0.0/16
service-cidr: 10.110.0.0/16
cluster-dns: 10.110.0.10
cni: none
disable:
  - rke2-ingress-nginx
  - rke2-canal
disable-kube-proxy: true
egress-selector-mode: disabled
protect-kernel-defaults: false

# Enable for RKE2 service

sudo systemctl enable rke2-server
sudo systemctl start rke2-server


# Join Worker Nodes

curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -


sudo mkdir -p /etc/rancher/rke2
sudo vim /etc/rancher/rke2/config.yaml

# Config

server: https://192.168.100.10:9345
token: "1234!@#$"


# Enable for RKE2 Agent service
# sudo dnf install rke2-agent (Sometimes need RK2 Package)

sudo systemctl enable rke2-agent
sudo systemctl start rke2-agent

### install l2 layer Announcement Policy

kubectl apply -f cilium-l2.yaml 

#### install cilium LB pool

kubectl apply -f cilium-lb-ip-pool.yaml  

### check Cilium IP pool again

kubectl get ciliumloadbalancerippool,ciliumpodippool -A  
