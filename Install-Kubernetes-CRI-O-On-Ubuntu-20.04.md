*** Guide install Kuberneter with CRI-O ***  
### Menu
[1. Prepare the environment](#1)  
[2. Install environment for all nodes Kubernetes ](#2)
- [2.1 Config hostname](#2.1)
- [2.2 Install required packages ](#2.2)
- [2.3 Install CRI-O](#2.3) 
- [2.4 Install kubelet, kubeadm and kubectl](#2.4) 

[3. Initialization Cluster](#3)
- [3.1 Initialization Cluster master nodes](#3.1)
- [3.2 Join Cluster worker nodes](#3.2)

[4. Containerd cheat sheet command](#4)

<a name="1"></a>
### 1. Prepare the environment
    - OS: Ubuntu-20.04
    - Kubelet vesion: v1.23.4
    - CRI-O version: 1.22.2

    Type    Hostname	Specs               IP Addess
    Master	k8s-m01     4GB Ram, 2vcpus     IP: 10.0.0.11/24
    Worker	k8s-w01     4GB Ram, 2vcpus     IP: 10.0.0.14/24
    Worker	k8s-w02     4GB Ram, 2vcpus     IP: 10.0.0.15/24

<a name="2"></a>
### 2. Install environment for all nodes Kubernetes
<a name="2.1"></a>
### 2.1 Config hostname
    cat <<EOF >>/etc/hosts
    10.0.0.11 k8s-m01
    10.0.0.14 k8s-w01
    10.0.0.15 k8s-w02
    EOF

<a name="2.2"></a>
### 2.2 Install required packages 
    apt install -y curl wget vim nano gnupg2 software-properties-common apt-transport-https ca-certificates

<a name="2.3"></a>
### 2.3 Install CRI-O
-  Configure persistent loading of modules

        cat <<EOF > /etc/modules-load.d/containerd.conf
        overlay
        br_netfilter
        EOF
-  Load at runtime

        sudo modprobe overlay
        sudo modprobe br_netfilter
- Check br_netfilter module running

        root@localhost:~# lsmod | grep br_netfilter
        br_netfilter           28672  0
        bridge                176128  1 br_netfilter


- Ensure sysctl params are set

        cat <<EOF >> /etc/sysctl.d/kubernetes.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        net.ipv4.ip_forward = 1
        EOF
- Reload configs

        sudo sysctl --system
- Add Cri-o repo

        sudo -s
        OS="xUbuntu_20.04"
        VERSION=1.22
        echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
        echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
        curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
        curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -
- Install CRI-O

        sudo apt update
        sudo apt install -y cri-o cri-o-runc
- Update CRI-O CIDR subnet

        sudo sed -i 's/10.85.0.0/192.168.0.0/g' /etc/cni/net.d/100-crio-bridge.conf

- Restart & enable cri-o service

        systemctl daemon-reload
        sudo systemctl restart crio
        sudo systemctl enable crio
        sudo systemctl status crio
    # Output
    ```sh
    root@localhost:~# sudo systemctl status crio
    ● crio.service - Container Runtime Interface for OCI (CRI-O)
        Loaded: loaded (/lib/systemd/system/crio.service; enabled; vendor preset: enabled)
        Active: active (running) since Sun 2022-03-06 06:29:57 UTC; 1s ago
        Docs: https://github.com/cri-o/cri-o
    Main PID: 46561 (crio)
        Tasks: 10
        Memory: 12.4M
        CGroup: /system.slice/crio.service
                └─46561 /usr/bin/crio

    Mar 06 06:29:56 k8s-m01 crio[46561]: time="2022-03-06 06:29:56.999336856Z" level=info msg="No seccomp profile specified, using the internal default"
    Mar 06 06:29:56 k8s-m01 crio[46561]: time="2022-03-06 06:29:56.999348745Z" level=info msg="Installing default AppArmor profile: crio-default"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.030926985Z" level=info msg="No blockio config file specified, blockio not configured"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.030990066Z" level=info msg="RDT not available in the host system"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.039426069Z" level=info msg="Found CNI network cbr0 (type=flannel) at /etc/cni/net.d/10-flannel.conflist"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.047225387Z" level=info msg="Found CNI network crio (type=bridge) at /etc/cni/net.d/100-crio-bridge.conf"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.049552153Z" level=info msg="Found CNI network 200-loopback.conf (type=loopback) at /etc/cni/net.d/200-loopback.conf"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.049594352Z" level=info msg="Updated default CNI network name to cbr0"
    Mar 06 06:29:57 k8s-m01 crio[46561]: time="2022-03-06 06:29:57.055260485Z" level=warning msg="Error encountered when checking whether cri-o should wipe images: version file /var/lib/crio/ve>
    Mar 06 06:29:57 k8s-m01 systemd[1]: Started Container Runtime Interface for OCI (CRI-O).
    ```

<a name="2.4"></a>
### 2.4 Install kubelet, kubeadm and kubectl

- Install kubelet, kubeadm and kubectl

        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
        sudo apt update
        sudo apt -y install kubelet kubeadm kubectl
        sudo apt-mark hold kubelet kubeadm kubectl
- Check version of kubectl

        kubectl version --client && kubeadm version
    # Output   
    ```sh
        root@localhost:~# kubectl version --client && kubeadm version
        Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:38:05Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}
        kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:36:57Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}
    ```
- Disable Swap

        sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
        sudo swapoff -a
- Enable kubelet service

        systemctl enable kubelet
<a name="3"></a>
### 3 Initialization Cluster

<a name="3.1"></a>
### 3.1 Initialization Cluster master nodes

- Initialization master

        kubeadm init --apiserver-advertise-address=10.0.0.11 --pod-network-cidr=10.244.0.0/16
    # Output
    ```sh
        root@k8s-m01:~# kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /var/run/crio/crio.sock
        [init] Using Kubernetes version: v1.23.4
        [preflight] Running pre-flight checks
        [preflight] Pulling images required for setting up a Kubernetes cluster
        [preflight] This might take a minute or two, depending on the speed of your internet connection
        [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
        [certs] Using certificateDir folder "/etc/kubernetes/pki"
        [certs] Generating "ca" certificate and key
        [certs] Generating "apiserver" certificate and key
        [certs] apiserver serving cert is signed for DNS names [k8s-m01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.11]
        [certs] Generating "apiserver-kubelet-client" certificate and key
        [certs] Generating "front-proxy-ca" certificate and key
        [certs] Generating "front-proxy-client" certificate and key
        [certs] Generating "etcd/ca" certificate and key
        [certs] Generating "etcd/server" certificate and key
        [certs] etcd/server serving cert is signed for DNS names [k8s-m01 localhost] and IPs [10.0.0.11 127.0.0.1 ::1]
        [certs] Generating "etcd/peer" certificate and key
        [certs] etcd/peer serving cert is signed for DNS names [k8s-m01 localhost] and IPs [10.0.0.11 127.0.0.1 ::1]
        [certs] Generating "etcd/healthcheck-client" certificate and key
        [certs] Generating "apiserver-etcd-client" certificate and key
        [certs] Generating "sa" key and public key
        [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
        [kubeconfig] Writing "admin.conf" kubeconfig file
        [kubeconfig] Writing "kubelet.conf" kubeconfig file
        [kubeconfig] Writing "controller-manager.conf" kubeconfig file
        [kubeconfig] Writing "scheduler.conf" kubeconfig file
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Starting the kubelet
        [control-plane] Using manifest folder "/etc/kubernetes/manifests"
        [control-plane] Creating static Pod manifest for "kube-apiserver"
        [control-plane] Creating static Pod manifest for "kube-controller-manager"
        [control-plane] Creating static Pod manifest for "kube-scheduler"
        [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
        [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
        [apiclient] All control plane components are healthy after 11.506063 seconds
        [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
        [kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
        NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
        [upload-certs] Skipping phase. Please see --upload-certs
        [mark-control-plane] Marking the node k8s-m01 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
        [mark-control-plane] Marking the node k8s-m01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
        [bootstrap-token] Using token: 4cy2cg.hmm6isy9fj1vkdex
        [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
        [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
        [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
        [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
        [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
        [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
        [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
        [addons] Applied essential addon: CoreDNS
        [addons] Applied essential addon: kube-proxy

        Your Kubernetes control-plane has initialized successfully!

        To start using your cluster, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

        Alternatively, if you are the root user, you can run:

        export KUBECONFIG=/etc/kubernetes/admin.conf

        You should now deploy a pod network to the cluster.
        Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        https://kubernetes.io/docs/concepts/cluster-administration/addons/

        Then you can join any number of worker nodes by running the following on each as root:

        kubeadm join 10.0.0.11:6443 --token 4cy2cg.hmm6isy9fj1vkdex \
                --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 
        root@k8s-m01:~# 
    ```
- You can optionally pass Socket file for runtime and advertise address depending on your setup.

        # Cri-o
        sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /var/run/crio/crio.sock

        # Containerd
        sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /run/containerd/containerd.sock

        # Docker
        sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /var/run/docker.sock

- Config environment runing kubeclt 

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
        export KUBECONFIG=/etc/kubernetes/admin.conf
        echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc

- Check status nodes

        kubectl get nodes 
        kubectl get nodes  -o wide
    # Output
    ```sh
    root@k8s-m01:~# kubectl get nodes
    NAME      STATUS   ROLES                  AGE    VERSION
    k8s-m01   Ready    control-plane,master   102s   v1.23.4
    root@k8s-m01:~# kubectl get nodes -o wide
    NAME      STATUS   ROLES                  AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   Ready    control-plane,master   107s   v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   cri-o://1.22.2


    ```
- Check status pods

        kubectl get pods --all-namespaces
        kubectl get pods --all-namespaces -o wide
    # Output
    ```sh
    root@k8s-m01:~# kubectl get pods --all-namespaces
    NAMESPACE     NAME                              READY   STATUS    RESTARTS      AGE
    kube-system   coredns-64897985d-hh56w           0/1     Pending   0             14m
    kube-system   coredns-64897985d-l4x2g           0/1     Pending   0             14m
    kube-system   etcd-k8s-m01                      1/1     Running   2 (10m ago)   14m
    kube-system   kube-apiserver-k8s-m01            1/1     Running   2 (10m ago)   14m
    kube-system   kube-controller-manager-k8s-m01   1/1     Running   2 (10m ago)   14m
    kube-system   kube-proxy-x7cns                  1/1     Running   1 (10m ago)   14m
    kube-system   kube-scheduler-k8s-m01            1/1     Running   2 (10m ago)   14m
    root@k8s-m01:~# kubectl get pods --all-namespaces -o wide
    NAMESPACE     NAME                              READY   STATUS    RESTARTS      AGE   IP          NODE      NOMINATED NODE   READINESS GATES
    kube-system   coredns-64897985d-hh56w           0/1     Pending   0             14m   <none>      <none>    <none>           <none>
    kube-system   coredns-64897985d-l4x2g           0/1     Pending   0             14m   <none>      <none>    <none>           <none>
    kube-system   etcd-k8s-m01                      1/1     Running   2 (10m ago)   14m   10.0.0.11   k8s-m01   <none>           <none>
    kube-system   kube-apiserver-k8s-m01            1/1     Running   2 (10m ago)   14m   10.0.0.11   k8s-m01   <none>           <none>
    kube-system   kube-controller-manager-k8s-m01   1/1     Running   2 (10m ago)   14m   10.0.0.11   k8s-m01   <none>           <none>
    kube-system   kube-proxy-x7cns                  1/1     Running   1 (10m ago)   14m   10.0.0.11   k8s-m01   <none>           <none>
    kube-system   kube-scheduler-k8s-m01            1/1     Running   2 (10m ago)   14m   10.0.0.11   k8s-m01   <none>           <none>
    root@k8s-m01:~# 
    ```
- Check status pods

        kubectl get pods --all-namespace
    # Output
    ```sh
    root@k8s-m01:~# kubectl get pods --all-namespaces
    NAMESPACE     NAME                              READY   STATUS    RESTARTS      AGE
    kube-system   coredns-64897985d-hh56w           1/1     Running   0             18m
    kube-system   coredns-64897985d-l4x2g           1/1     Running   0             18m
    kube-system   etcd-k8s-m01                      1/1     Running   2 (14m ago)   18m
    kube-system   kube-apiserver-k8s-m01            1/1     Running   2 (14m ago)   18m
    kube-system   kube-controller-manager-k8s-m01   1/1     Running   2 (14m ago)   18m
    kube-system   kube-flannel-ds-bxqcp             1/1     Running   0             77s
    kube-system   kube-proxy-x7cns                  1/1     Running   1 (14m ago)   18m
    kube-system   kube-scheduler-k8s-m01            1/1     Running   2 (14m ago)   18m
    root@k8s-m01:~# 
    ```
- Create token join cluster

        kubeadm token create --print-join-command    
    # Output
    ```sh
    root@k8s-m01:~# kubeadm token create --print-join-command
    kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 
    root@k8s-m01:~# 
    ```

<a name="3.2"></a>
### 3.2 Join Cluster worker nodes
- Worker node 1

        kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 --cri-socket /var/run/crio/crio.sock
    # Output
    ```sh
    root@k8s-w01:~# kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 --cri-socket /var/run/crio/crio.sock
    [preflight] Running pre-flight checks
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    W0305 19:46:53.842434    4649 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

    root@k8s-w01:~# 

    ```
- Worker node 2

        kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 --cri-socket /var/run/crio/crio.sock
    # Output
    ```sh
    root@k8s-w02:~# kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 --cri-socket /var/run/crio/crio.sock
    [preflight] Running pre-flight checks
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    W0305 19:49:42.091365    1907 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

    root@k8s-w02:~# 

    ```
- Check status nodes on master node

        kubectl get nodes
        kubectl get nodes -o wide
    # Output
    ```sh
    root@k8s-m01:~# kubectl get nodes
    NAME      STATUS   ROLES                  AGE     VERSION
    k8s-m01   Ready    control-plane,master   11m     v1.23.4
    k8s-w01   Ready    <none>                 4m54s   v1.23.4
    k8s-w02   Ready    <none>                 3m48s   v1.23.4
    root@k8s-m01:~# kubectl get nodes -o wide
    NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   Ready    control-plane,master   11m     v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   cri-o://1.22.2
    k8s-w01   Ready    <none>                 4m56s   v1.23.4   10.0.0.14     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   cri-o://1.22.2
    k8s-w02   Ready    <none>                 3m50s   v1.23.4   10.0.0.15     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   cri-o://1.22.2

 