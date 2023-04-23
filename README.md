#### 0. Delete cluster 

```bash
/usr/local/bin/k3s-uninstall.sh
```

```bash 
rm -rf /etc/rancher/*
```

```bash
rm -rf /var/lib/rancher/
```

#### 1. Ctr pakages:

```bash
zypper in -y docker containerd-ctr nfs-client nfs-kernel-server xfsprogs ceph-common open-iscsi cloud-init clone-master-clean-up
```
#### 2. Enable forwarding:

```bash
sysctl net.ipv4.conf.all.forwarding=1
```

```bash
sysctl net.ipv6.conf.all.forwarding=1
```

#### 3. System settings:

```bash
vi /etc/ssh/sshd_config
```

```bash
PubkeyAuthentication yes
```

```bash
AllowTcpForwarding yes
```

```bash
vi /etc/sysctl.conf
```

```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.ip_forward = 1
```

```bash
vi /etc/sysctl.d/90-rancher.conf
```

```bash
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
net.bridge.bridge-nf-call-iptables=1
```

```bash
vi /etc/modules-load.d/modules-rancher.conf 
```

```bash
br_netfilter
ip6_udp_tunnel
ip_set_hash_ip
ip_set_hash_net
iptable_filter
iptable_nat
iptable_mangle
iptable_raw
nf_conntrack_netlink
nf_conntrack
nf_conntrack_ipv4
nf_defrag_ipv4
nf_nat
nf_nat_ipv4
nf_nat_masquerade_ipv4
udp_tunnel
veth
vxlan
xt_addrtype
xt_conntrack
xt_comment
xt_mark
xt_multiport
xt_nat
xt_recent
xt_set
xt_statistic
xt_tcpudp
```

#### 4. Set ip address

```bash
export VIP=<Vip>
```

#### 5. Set interface

```bash
export INTERFACE=eth0
```

#### 6. Set version kube-vip

```bash
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
```
#### 7. Manual set

```bash
export KVVERSION=v0.5.0
```

#### 8. Pull image and render manifest 

```bash
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

```bash
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection
```

#### 9. Copy manifest to

```bash
~/kube-vip.yaml
```

#### 10. install k3s 

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.23.8+k3s2 sh -s - server --cluster-init --tls-san $VIP
```

```bash
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
```

```bash
kubectl apply -f ~/kube-vip.yaml
```

#### 11. Join cluster

```bash
cat /var/lib/rancher/k3s/server/token
```

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - server --server https://#VIP:6443
```
#### 12. install helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### 13. add repo to helm

```bash
helm repo add jetstack https://charts.jetstack.io
```

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

```bash
helm repo update
```
#### 14. install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
--create-namespace \
--namespace cert-manager \
--version v1.9.1 \
--set installCRDs=true \
--set nodeSelector."kubernetes\.io/os"=linux \
--set webhook.nodeSelector."kubernetes\.io/os"=linux \
--set cainjector.nodeSelector."kubernetes\.io/os"=linux \
--set startupapicheck.nodeSelector."kubernetes\.io/os"=linux
```  
  
#### 15. install Rancher

```bash
helm install rancher rancher-latest/rancher \
--create-namespace \
--namespace cattle-system \
--set hostname=rancher.$VIP.sslip.io \
--set replicas=3 \
--set bootstrapPassword=<PASSWORD_FOR_RANCHER_ADMIN>
```
