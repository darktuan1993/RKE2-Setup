# Cụm Kubernetes HA với RKE2
## Giới thiệu

RKE2, còn được gọi là RKE Government, là phân phối Kubernetes thế hệ tiếp theo của Rancher. Đây là một phân phối Kubernetes hoàn toàn tuân thủ, tập trung vào bảo mật và tuân thủ trong khu vực chính phủ liên bang của Hoa Kỳ.

## Khác biệt so với RKE như thế nào?

RKE2 không dựa vào Docker như RKE1. RKE1 đã sử dụng Docker để triển khai và quản lý các thành phần của control plane (chúng được triển khai dưới dạng container, không phải pod, và được quản lý bởi Docker) cũng như runtime container cho Kubernetes. RKE2 khởi chạy các thành phần của control plane dưới dạng các pod tĩnh, được quản lý bởi kubelet. Runtime container được tích hợp là containerd.

RKE2 trước đây được gọi là RKE Government vì nó ban đầu được thiết kế để triển khai trong các dự án của chính phủ Hoa Kỳ. Như bạn có thể tưởng tượng, bảo mật là điều cực kỳ quan trọng trong các tình huống này, vì vậy RKE2 tập trung vào bảo mật và tuân thủ ngay từ đầu, trong khi RKE là một phân phối Kubernetes tiêu chuẩn mà không chú trọng đặc biệt vào bảo mật.

## Mô hình cài đặt Cluster Rancher-RKE2-HA
[![N|Solid](https://lh3.googleusercontent.com/pw/AP1GczMq58cxkGi2VLLpJ0CL3-J2iuy70Tb0_uKtRq_P770Xz8IN9zVAowWKNIQklt6YkXd3tPWiqMUiwpmqhUT2t2h-XoRYRnQ0JZfKYHQj2R81gLCQYsg=w2400)]


Cụm cài đặt 3 master - 3 slaves

## Config host cho tất cả các node /etc/hosts

``` bash
cat /etc/hosts
```

cấu hình host nội bộ theo domain trỏ vào ip của nginx với 2 cấu hình TCP bao gồm
load balancing TCP port rke2-server 9345 và load balancing TCP port api-server

## Cài đặt Rancher RKE2
### Setup cơ bản cho tất cả các máy trong cluster :
``` bash
disable the firewall and swap
sudo systemctl stop ufw
sudo systemctl disable ufw
apt-get update && apt-get upgrade -y
apt-get install -y vim
apt install curl apt-transport-https vim git nfs-common wget software-properties-common lsb-release ca-certificates -y
swapoff -a; sed -i '/swap/d' /etc/fstab
modprobe overlay
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 'br_netfilter' > /etc/modules-load.d/k8s.conf

{
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.ip_local_port_range = 10240 65535
vm.swappiness = 10
vm.overcommit_memory = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
fs.file-max = 500000
fs.inotify.max_user_watches = 524288
kernel.sched_migration_cost_ns=500000
kernel.sched_autogroup_enabled=0
EOF
}
```
list version:
https://docs.rke2.io/release-notes/v1.28.X

### CÀI ĐẶT CHO NODE MASTER:
``` bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.28.15+rke2r1 INSTALL_RKE2_TYPE=server sh -
```

````bash
mkdir -p /etc/rancher/rke2
touch /etc/rancher/rke2/config.yaml
mkdir -p /var/backups/etcd-snapshot
````
#### Với node master đầu tiên
````bash
echo "write-kubeconfig-mode: 0600
tls-san:
  - <domain của bạn >
  - <ip-máy chủ load balancing / ip-node master>
disable-kube-proxy: no
disable:
  - rke2-ingress-nginx
disable-cloud-controller: yes
#### ETCD ####
etcd-snapshot-name: etcd-snapshot
etcd-snapshot-schedule-cron: \"0 12 * * *\"
etcd-snapshot-retention: 5
etcd-snapshot-dir: /var/backups/etcd-snapshot
etcd-snapshot-compress: true


#### KUBELET ####
kubelet-arg:
- kube-reserved=cpu=400m,memory=2Gi
- system-reserved=cpu=400m,memory=2Gi" >> /etc/rancher/rke2/config.yaml
````

#### Đối với node master khác

````bash
### Node Master Other ###
echo "
server: https://<ip-máy chủ load balancing / ip-node master>:9345
token: <token join cluster>
write-kubeconfig-mode: 0600
tls-san:
  - <domain của bạn >
  - <ip-máy chủ load balancing / ip-node master>
disable-kube-proxy: no
disable:
  - rke2-ingress-nginx
disable-cloud-controller: yes
#### ETCD ####
etcd-snapshot-name: etcd-snapshot
etcd-snapshot-schedule-cron: "0 12 * * *"
etcd-snapshot-retention: 5
etcd-snapshot-dir: /var/backups/etcd-snapshot
etcd-snapshot-compress: true


#### KUBELET ####
kubelet-arg:
- kube-reserved=cpu=400m,memory=1Gi
- system-reserved=cpu=400m,memory=1Gi" >> /etc/rancher/rke2/config.yaml

````

```` bash
systemctl enable rke2-server.service
systemctl start rke2-server.service
````

############################################################

### CÀI ĐẶT CHO NODE WOKER KHÁC :

````bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.30.6+rke2r1 INSTALL_RKE2_TYPE=agent sh -

echo "
server: https://<ip-máy chủ load balancing / ip-node master>:9345
token: <token join cluster>
write-kubeconfig-mode: 0600

#### KUBELET ####
kubelet-arg:
- kube-reserved=cpu=800m,memory=2Gi
- system-reserved=cpu=800m,memory=2Gi" >> /etc/rancher/rke2/config.yaml

````


Sau đó sẽ setup các manifest khác