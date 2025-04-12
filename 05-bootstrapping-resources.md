# Bootstrapping Resources

- Kubernetes components are stateless and store cluster state in etcd. 
- Purpose: bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: controller-0, controller-1, and controller-2.

1. Output the ssh commands with the IP address of the contoller nodes. 
    ```
    for instance in controller-0 controller-1 controller-2; do
      external_ip=$(aws ec2 describe-instances --filters \
        "Name=tag:Name,Values=${instance}" \
        "Name=instance-state-name,Values=running" \
        --output text --query 'Reservations[].Instances[].PublicIpAddress')

      echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
    done
    ```

2. SSH into each one of the IP addresses received in last step. 
    ```
    ssh -i kubernetes.id_rsa ubuntu@54.248.147.5
    ssh -i kubernetes.id_rsa ubuntu@18.181.145.51
    ssh -i kubernetes.id_rsa ubuntu@43.207.89.126 
    ```

3. Run Commands in tmux
    * Configure to run commands simultaneously
        * CTRL + B, :, setw synchronize-panes
    
## Bootstrapping an etcd Cluster Member
1. Download etcd binaries
    ```
    wget -q --show-progress --https-only --timestamping "https://github.com/etcd-io/etcd/releases/download/v3.5.8/etcd-v3.5.8-linux-amd64.tar.gz"
    ```
2. Extract and install the etcd server and the etcdctl command line utility:
    ```
    tar -xvf etcd-v3.5.8-linux-amd64.tar.gz
    sudo mv etcd-v3.5.8-linux-amd64/etcd* /usr/local/bin/
    ```
3. Configure the etcd Server
    ```
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo chmod 700 /var/lib/etcd
    sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
    ```
4. The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:
    ```
    INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
    ```
    > 10.0.1.10, 10.0.1.11, 10.0.1.12 
5. Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance: 
    ```
    ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
    echo "${ETCD_NAME}"
    ```
    > controller-0, controller-1, controller-2 
6. Create the etcd.service systemd unit file:
    ```
    cat <<EOF | sudo tee /etc/systemd/system/etcd.service
    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos

    [Service]
    Type=notify
    ExecStart=/usr/local/bin/etcd \\
      --name ${ETCD_NAME} \\
      --cert-file=/etc/etcd/kubernetes.pem \\
      --key-file=/etc/etcd/kubernetes-key.pem \\
      --peer-cert-file=/etc/etcd/kubernetes.pem \\
      --peer-key-file=/etc/etcd/kubernetes-key.pem \\
      --trusted-ca-file=/etc/etcd/ca.pem \\
      --peer-trusted-ca-file=/etc/etcd/ca.pem \\
      --peer-client-cert-auth \\
      --client-cert-auth \\
      --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
      --advertise-client-urls https://${INTERNAL_IP}:2379 \\
      --initial-cluster-token etcd-cluster-0 \\
      --initial-cluster controller-0=https://10.0.1.10:2380,controller-1=https://10.0.1.11:2380,controller-2=https://10.0.1.12:2380 \\
      --initial-cluster-state new \\
      --data-dir=/var/lib/etcd
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
7. Start the etcd Server
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd
    ```
    > Created symlink /etc/systemd/system/multi-user.target.wants/etcd.service → /etc/systemd/system/etcd.service. 

8. Verification
    List the etcd cluster members:
    ```
    sudo ETCDCTL_API=3 etcdctl member list \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/etcd/ca.pem \
      --cert=/etc/etcd/kubernetes.pem \
      --key=/etc/etcd/kubernetes-key.pem
    ```
    Output
    ```
    bbeedf10f5bbaa0c, started, controller-2, https://10.0.1.12:2380, https://10.0.1.12:2379, false
    eecdfcb7e79fc5dd, started, controller-1, https://10.0.1.11:2380, https://10.0.1.11:2379, false
    f9b0e395cb8278dc, started, controller-0, https://10.0.1.10:2380, https://10.0.1.10:2379, false
    ```

## Control Plane

1. Create the Kubernetes configuration directory
    ```
    sudo mkdir -p /etc/kubernetes/config
    ```
2. Download the official Kubernetes release binaries
    ```
    wget -q --show-progress --https-only --timestamping \
      "https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kube-apiserver" \
      "https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kube-controller-manager" \
      "https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kube-scheduler" \
      "https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kubectl"
    ```
3. Install the Kubernetes binaries
    ```
    chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
    sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
    ```
4. Configure the Kubernetes API Server
    ```
    sudo mkdir -p /var/lib/kubernetes/

    sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
      service-account-key.pem service-account.pem \
      encryption-config.yaml /var/lib/kubernetes/
    ```
5. The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:
    ```
    INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
    ```
6. Create kube-apiserver.service systemd unit file 
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
    [Unit]
    Description=Kubernetes API Server
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-apiserver \\
      --advertise-address=${INTERNAL_IP} \\
      --allow-privileged=true \\
      --apiserver-count=3 \\
      --audit-log-maxage=30 \\
      --audit-log-maxbackup=3 \\
      --audit-log-maxsize=100 \\
      --audit-log-path=/var/log/audit.log \\
      --authorization-mode=Node,RBAC \\
      --bind-address=0.0.0.0 \\
      --client-ca-file=/var/lib/kubernetes/ca.pem \\
      --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
      --etcd-cafile=/var/lib/kubernetes/ca.pem \\
      --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
      --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
      --etcd-servers=https://10.0.1.10:2379,https://10.0.1.11:2379,https://10.0.1.12:2379 \\
      --event-ttl=1h \\
      --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
      --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
      --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
      --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
      --runtime-config='api/all=true' \\
      --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
      --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
      --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \\
      --service-cluster-ip-range=10.32.0.0/24 \\
      --service-node-port-range=30000-32767 \\
      --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
      --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
7. Configure the Kubernetes Controller Manager

    Move the kube-controller-manager kubeconfig into place:
    ```
    sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
    ```
    Create the kube-controller-manager.service systemd unit file:
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
    [Unit]
    Description=Kubernetes Controller Manager
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-controller-manager \\
      --bind-address=0.0.0.0 \\
      --cluster-cidr=10.200.0.0/16 \\
      --cluster-name=kubernetes \\
      --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
      --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
      --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
      --leader-elect=true \\
      --root-ca-file=/var/lib/kubernetes/ca.pem \\
      --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
      --service-cluster-ip-range=10.32.0.0/24 \\
      --use-service-account-credentials=true \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
8. Configure the Kubernetes Scheduler
    Move the kube-scheduler kubeconfig into place:
    ```
    sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
    ```
    Create the kube-scheduler.yaml configuration file 
    ```
    cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
        apiVersion: kubescheduler.config.k8s.io/v1beta3
        kind: KubeSchedulerConfiguration
        clientConnection:
          kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
        leaderElection:
          leaderElect: true
        EOF
        ```
        Create the kube-scheduler.service systemd unit file 
        ```
        cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
    [Unit]
    Description=Kubernetes Scheduler
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-scheduler \\
      --config=/etc/kubernetes/config/kube-scheduler.yaml \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
9. Start the Controller Services
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
    sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
    ```
    Output
    ```    
    Created symlink /etc/systemd/system/multi-user.target.wants/kube-apiserver.service → /etc/systemd/system/kube-apiserver.service.
    Created symlink /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service → /etc/systemd/system/kube-controller-manager.service.
    Created symlink /etc/systemd/system/multi-user.target.wants/kube-scheduler.service → /etc/systemd/system/kube-scheduler.service.
    ```
    > Allow up to 10 seconds for the Kubernetes API Server to fully initialize. 
10. Verification
    ```
    kubectl cluster-info --kubeconfig admin.kubeconfig
    ```
    ```
    Kubernetes control plane is running at https://127.0.0.1:6443
    ```
11. Add Host File Entries
    In order for kubectl exec commands to work, the controller nodes must each be able to resolve the worker hostnames (not set up by default in AWS).
    * Add manual host entries on each of the controller nodes
        ```
        cat <<EOF | sudo tee -a /etc/hosts
        10.0.1.20 ip-10-0-1-20
        10.0.1.21 ip-10-0-1-21
        10.0.1.22 ip-10-0-1-22
        EOF
        ```
12. RBAC for Kubelet Authorization
    configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.
    The commands in this section will effect the entire cluster and only need to be run once from one of the controller nodes.
    * Create the system:kube-apiserver-to-kubelet ClusterRole with permissions to access the Kubelet API and perform most common tasks associated with managing pods 
    ```
    cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
      name: system:kube-apiserver-to-kubelet
    rules:
      - apiGroups:
          - ""
        resources:
          - nodes/proxy
          - nodes/stats
          - nodes/log
          - nodes/spec
          - nodes/metrics
        verbs:
          - "*"
    EOF
    ```
    > The Kubernetes API Server authenticates to the Kubelet as the kubernetes user using the client certificate as defined by the --kubelet-client-certificate flag. 
    Output
    ```
    clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
    ```
    * Bind the system:kube-apiserver-to-kubelet ClusterRole to the kubernetes user 
    ```
    cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: system:kube-apiserver
      namespace: ""
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:kube-apiserver-to-kubelet
    subjects:
      - apiGroup: rbac.authorization.k8s.io
        kind: User
        name: kubernetes
    EOF
    ```
    Output
    ```
    clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
    ```
13. Verification of cluster public endpoint
    * Exit from SSH if haven't. Run the following commands from the same machine used to create the compute instances. The compute instances created in this tutorial will not have permission to complete this section. 
    * Retrieve the kubernetes-the-hard-way Load Balancer address 
        ```
        KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
      --load-balancer-arns ${LOAD_BALANCER_ARN} \
      --output text --query 'LoadBalancers[].DNSName')
        ```
        > kubernetes-f28a31d1f2c2881b.elb.ap-northeast-1.amazonaws.com
    * Make a HTTP request for the Kubernetes version info 
        ```
        curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}/version
        ```
        Output
        ```
        {
            "major": "1","
            minor": "28",
            "gitVersion": "v1.28.3",
            "gitCommit": "a8a1abc25cad87333840cd7d54be2efaf31a3177",
            "gitTreeState": "clean",
            "buildDate": "2023-10-18T11:33:18Z",
            "goVersion": "go1.20.10",
            "compiler": "gc",
            "platform": "linux/amd64"
        }
        ```
        
## Kubernetes Worker Nodes
In this lab you will bootstrap three Kubernetes worker nodes. 
The following components will be installed on each node: runc, container networking plugins, containerd, kubelet, and kube-proxy.

SSH to worker instance
```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  echo ssh -i kubernetes.id_rsa ubuntu@$external_ip
done
```

1. Install the OS dependencies:
    ```
    sudo apt-get update
    sudo apt-get -y install socat conntrack ipset
    ```
    > The socat binary enables support for the kubectl port-forward command.
2. Disable Swap
    By default the kubelet will fail to start if swap is enabled. 
    * Verification
        ```
        sudo swapon --show
        ```
    * Disabling
        ```
        sudo swapoff -a
        ```
3. Download and Install Worker Binaries
    ```
    wget -q --show-progress --https-only --timestamping \
      https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz \
      https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64 \
      https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz \
      https://github.com/containerd/containerd/releases/download/v1.7.7/containerd-1.7.7-linux-amd64.tar.gz \
      https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kubectl \
      https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kube-proxy \
      https://dl.k8s.io/release/v1.28.3/bin/linux/amd64/kubelet
    ```
4. Create the installation directories:
    ```
    sudo mkdir -p \
      /etc/cni/net.d \
      /opt/cni/bin \
      /var/lib/kubelet \
      /var/lib/kube-proxy \
      /var/lib/kubernetes \
      /var/run/kubernetes
    ```
5. Install the worker binaries:
    ```
    mkdir containerd
    tar -xvf crictl-v1.28.0-linux-amd64.tar.gz
    tar -xvf containerd-1.7.7-linux-amd64.tar.gz -C containerd
    sudo tar -xvf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin/
    sudo mv runc.amd64 runc
    chmod +x crictl kubectl kube-proxy kubelet runc 
    sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
    sudo mv containerd/bin/* /bin/
    ```
6. Configure CNI Networking
    * Retrieve the Pod CIDR range for the current compute instance (exit from SSH if haven't)
        ```
        POD_CIDR=$(curl -s http://169.254.169.254/latest/user-data/ \
          | tr "|" "\n" | grep "^pod-cidr" | cut -d"=" -f2)
        echo "${POD_CIDR}"
        ```
        > 10.200.0.0.0/24, 10.200.0.1.0/24, 10.200.0.2.0/24
    * Create the bridge network configuration file
        ```
        cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
        {
            "cniVersion": "1.3.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cnio0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
                "type": "host-local",
                "ranges": [
                  [{"subnet": "${POD_CIDR}"}]
                ],
                "routes": [{"dst": "0.0.0.0/0"}]
            }
        }
        EOF
        ```
    * Create the loopback network configuration file 
        ```
        cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
        {
            "cniVersion": "1.3.0",
            "name": "lo",
            "type": "loopback"
        }
        EOF
        ```
7. Configure containerd
    * Create the containerd configuration file 
        ```
        sudo mkdir -p /etc/containerd/
        ```
        ```
        cat << EOF | sudo tee /etc/containerd/config.toml
        [plugins]
          [plugins.cri.containerd]
            snapshotter = "overlayfs"
            [plugins.cri.containerd.default_runtime]
              runtime_type = "io.containerd.runtime.v1.linux"
              runtime_engine = "/usr/local/bin/runc"
              runtime_root = ""
        EOF
        ```
    * Create the containerd.service systemd unit file 
        ```
        cat <<EOF | sudo tee /etc/systemd/system/containerd.service
        [Unit]
        Description=containerd container runtime
        Documentation=https://containerd.io
        After=network.target

        [Service]
        ExecStartPre=/sbin/modprobe overlay
        ExecStart=/bin/containerd
        Restart=always
        RestartSec=5
        Delegate=yes
        KillMode=process
        OOMScoreAdjust=-999
        LimitNOFILE=1048576
        LimitNPROC=infinity
        LimitCORE=infinity

        [Install]
        WantedBy=multi-user.target
        EOF
        ```
8. Configure the Kubelet
    ```
    WORKER_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
    | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
    echo "${WORKER_NAME}"

    sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
    sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
    sudo mv ca.pem /var/lib/kubernetes/
    ```
9. Create the kubelet-config.yaml configuration file 
    ```
    cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    authentication:
      anonymous:
        enabled: false
      webhook:
        enabled: true
      x509:
        clientCAFile: "/var/lib/kubernetes/ca.pem"
    authorization:
      mode: Webhook
    clusterDomain: "cluster.local"
    clusterDNS:
      - "10.32.0.10"
    podCIDR: "${POD_CIDR}"
    resolvConf: "/run/systemd/resolve/resolv.conf"
    runtimeRequestTimeout: "15m"
    tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
    tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
    EOF
    ```
    > The resolvConf configuration is used to avoid loops when using CoreDNS for service discovery on systems running systemd-resolved.
10. Create the kubelet.service systemd unit file:
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
    [Unit]
    Description=Kubernetes Kubelet
    Documentation=https://github.com/kubernetes/kubernetes
    After=containerd.service
    Requires=containerd.service

    [Service]
    ExecStart=/usr/local/bin/kubelet \\
      --config=/var/lib/kubelet/kubelet-config.yaml \\
      --kubeconfig=/var/lib/kubelet/kubeconfig \\
      --register-node=true \\
      --v=2
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
11. Configure the Kubernetes Proxy
    ```
    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
    ```
12. Create the kube-proxy-config.yaml configuration file 
    ```
    cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
    kind: KubeProxyConfiguration
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    clientConnection:
      kubeconfig: "/var/lib/kube-proxy/kubeconfig"
    mode: "iptables"
    clusterCIDR: "10.200.0.0/16"
    EOF
    ```
13. Create the kube-proxy.service systemd unit file 
    ```
    cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
    [Unit]
    Description=Kubernetes Kube Proxy
    Documentation=https://github.com/kubernetes/kubernetes

    [Service]
    ExecStart=/usr/local/bin/kube-proxy \\
      --config=/var/lib/kube-proxy/kube-proxy-config.yaml
    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
14. Start the Worker Services
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable containerd kubelet kube-proxy
    sudo systemctl start containerd kubelet kube-proxy
    ```
15. Verification: List the registered Kubernetes nodes (exit SSH)
    ```
    external_ip=$(aws ec2 describe-instances --filters \
        "Name=tag:Name,Values=controller-0" \
        "Name=instance-state-name,Values=running" \
        --output text --query 'Reservations[].Instances[].PublicIpAddress')

    ssh -i kubernetes.id_rsa ubuntu@${external_ip} kubectl get nodes --kubeconfig admin.kubeconfig
    ```
    Output
    ```
    NAME           STATUS   ROLES    AGE   VERSION     
    ip-10-0-1-20   Ready    <none>   72s   v1.28.3
    ip-10-0-1-21   Ready    <none>   72s   v1.28.3
    ip-10-0-1-22   Ready    <none>   72s   v1.28.3 
    ```
    