# RKE2 Setup
## Đặt /etc/host

     cat /etc/hosts
     192.168.18.190 rke2-master.local
     192.168.18.191 rke2-slaves1.local
     192.168.18.189 rke2-slaves2.local
## Features
    # Master
    curl -sfL https://get.rke2.io | sh -

    # Slaves
    curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

    systemctl enable rke2-server.service
    systemctl start rke2-server.service
    journalctl -u rke2-server -f

## Truy cập cụm

    cp /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/
    chmod +x /usr/local/bin/kubectl
    export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
    
## Add node vào cụm tương tự vào các cụm

    cat /var/lib/rancher/rke2/server/node-token
    systemctl enable rke2-agent.service
    mkdir -p /etc/rancher/rke2/
    vim /etc/rancher/rke2/config.yaml
Config nội dung giống như sau

    server: https://<server>:9345
    token: <token from server node>
Sau đó restart lại service

    systemctl start rke2-agent.service
    
