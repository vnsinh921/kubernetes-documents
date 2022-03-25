*** Guide install Kuberneter with Containerd ***  
### Menu
[1. Prepare the environment](#1)  
[2. Install environment for all nodes Kubernetes ](#2)
- [2.1 Config hostname](#2.1)
- [2.2 Install required packages ](#2.2)
- [2.3 Install Containerd](#2.3) 
- [2.4 Install kubelet, kubeadm and kubectl](#2.4) 

[3. Initialization Cluster](#3)
- [3.1 Initialization Cluster master nodes](#3.1)
- [3.2 Join Cluster worker nodes](#3.2)

[4. Containerd cheat sheet command](#4)

<a name="1"></a>
### 1. Prepare the environment
    - OS: Ubuntu-20.04
    - Kubelet vesion: v1.23.4
    - Containerd version: 1.5.5

    Type    Hostname	Specs               IP Addess
    Master	k8s-m01     4GB Ram, 2vcpus     IP: 10.0.0.11/24
    Worker	k8s-w01     4GB Ram, 2vcpus     IP: 10.0.0.14/24
    Worker	k8s-w02     4GB Ram, 2vcpus     IP: 10.0.0.15/24

<a name="2"></a>
### 2. Install environment for all nodes Kubernetes
<a name="2.1"></a>
### 2.1 Config hostname
    ```sh
    cat <<EOF >>/etc/hosts
    10.0.0.11 k8s-m01
    10.0.0.14 k8s-w01
    10.0.0.15 k8s-w02
    EOF
    ```
<a name="2.2"></a>
### 2.2 Install required packages 
    ```sh
    apt install -y curl wget vim nano gnupg2 software-properties-common apt-transport-https ca-certificates
    ```
<a name="2.3"></a>
### 2.3 Install Containerd
-  Configure persistent loading of modules
    ```sh
    cat <<EOF > /etc/modules-load.d/containerd.conf
    overlay
    br_netfilter
    EOF
    ```
-  Load at runtime
    ```sh
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```
- Check br_netfilter module running
    ```sh
    root@localhost:~# lsmod | grep br_netfilter
    br_netfilter           28672  0
    bridge                176128  1 br_netfilter
    ```

- Ensure sysctl params are set
    ```sh
    cat <<EOF >> /etc/sysctl.d/kubernetes.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    ```
- Reload configs
    ```sh
    sudo sysctl --system
    ```
- Install containerd
    ```sh
    sudo apt update
    sudo apt install -y containerd
    ```
- Configure containerd
    ```sh
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    ```
    # Output
    ```sh
    disabled_plugins = []
    imports = []
    oom_score = 0
    plugin_dir = ""
    required_plugins = []
    root = "/var/lib/containerd"
    state = "/run/containerd"
    version = 2

    [cgroup]
    path = ""

    [debug]
    address = ""
    format = ""
    gid = 0
    level = ""
    uid = 0

    [grpc]
    address = "/run/containerd/containerd.sock"
    gid = 0
    max_recv_message_size = 16777216
    max_send_message_size = 16777216
    tcp_address = ""
    tcp_tls_cert = ""
    tcp_tls_key = ""
    uid = 0

    [metrics]
    address = ""
    grpc_histogram = false

    [plugins]

    [plugins."io.containerd.gc.v1.scheduler"]
        deletion_threshold = 0
        mutation_threshold = 100
        pause_threshold = 0.02
        schedule_delay = "0s"
        startup_delay = "100ms"

    ---

    [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
        accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
        args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
        env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
        path = "ctd-decoder"
        returns = "application/vnd.oci.image.layer.v1.tar+gzip"

    [timeouts]
    "io.containerd.timeout.shim.cleanup" = "5s"
    "io.containerd.timeout.shim.load" = "5s"
    "io.containerd.timeout.shim.shutdown" = "3s"
    "io.containerd.timeout.task.state" = "2s"

    [ttrpc]
    address = ""
    gid = 0
    uid = 0
    ```
- Config cgroup driver
    ```sh
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
    ```
- Restart containerd và enable containerd
    ```sh
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    sudo systemctl status containerd
    ```
    # Output
    ```sh

    root@localhost:~# systemctl status containerd
    ● containerd.service - containerd container runtime
        Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
        Active: active (running) since Sat 2022-03-05 19:04:21 UTC; 10s ago
        Docs: https://containerd.io
    Main PID: 2995 (containerd)
        Tasks: 11
        Memory: 23.5M
        CGroup: /system.slice/containerd.service
                └─2995 /usr/bin/containerd

    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.846530137Z" level=info msg=serving... address=/run/containerd/containerd.sock.ttrpc
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.847108968Z" level=info msg=serving... address=/run/containerd/containerd.sock
    Mar 05 19:04:21 localhost systemd[1]: Started containerd container runtime.
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.847827697Z" level=info msg="containerd successfully booted in 0.042172s"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.848456309Z" level=info msg="Start subscribing containerd event"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.849007609Z" level=info msg="Start recovering state"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.849104811Z" level=info msg="Start event monitor"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.849309617Z" level=info msg="Start snapshots syncer"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.849355225Z" level=info msg="Start cni network conf syncer"
    Mar 05 19:04:21 localhost containerd[2995]: time="2022-03-05T19:04:21.849366438Z" level=info msg="Start streaming server"

    ```

<a name="2.4"></a>
### 2.4 Install kubelet, kubeadm and kubectl

- Install kubelet, kubeadm and kubectl
    ```sh
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt update
    sudo apt -y install kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
- Check version of kubectl
    ```sh
    kubectl version --client && kubeadm version
    ```
    # Output   
    ```sh
    root@localhost:~# kubectl version --client && kubeadm version
    Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:38:05Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}
    kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:36:57Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}
    ```
- Disable Swap
    ```sh
    sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
    sudo swapoff -a
    ```
- Enable kubelet service
    ```sh
    systemctl enable kubelet
    ```
<a name="3"></a>
### 3 Initialization Cluster

<a name="3.1"></a>
### 3.1 Initialization Cluster master nodes

- Initialization master
    ```sh
    kubeadm init --apiserver-advertise-address=10.0.0.11 --pod-network-cidr=10.244.0.0/16
    ```
    # Output
    ```sh
        root@k8s-m01:~# kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16
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
    ```sh
    # Docker
    sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /var/run/docker.sock

    # Containerd
    sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /run/containerd/containerd.sock

    # Cri-o
    sudo kubeadm init --apiserver-advertise-address 10.0.0.11 --pod-network-cidr=10.244.0.0/16 --cri-socket /var/run/crio/crio.sock
    ```
- Config environment runing kubeclt 
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    export KUBECONFIG=/etc/kubernetes/admin.conf
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc
    ```
- Check status nodes
    ```sh
    kubectl get nodes 
    kubectl get nodes  -o wide
    ```
    # Output
    ```sh
    root@k8s-m01:~# kubectl get nodes 
    NAME      STATUS     ROLES                  AGE   VERSION
    k8s-m01   NotReady   control-plane,master   11m   v1.23.4
    root@k8s-m01:~# kubectl get nodes  -o wide
    NAME      STATUS     ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   NotReady   control-plane,master   11m   v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   containerd://1.5.5

    ```
- Check status pods
    ```sh
    kubectl get pods --all-namespaces
    kubectl get pods --all-namespaces -o wide
    ```
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
- Install a Network Plugin in the Plugin
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```
    # Output
    ```sh
    root@k8s-m01:~# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
    podsecuritypolicy.policy/psp.flannel.unprivileged created
    clusterrole.rbac.authorization.k8s.io/flannel created
    clusterrolebinding.rbac.authorization.k8s.io/flannel created
    serviceaccount/flannel created
    configmap/kube-flannel-cfg created
    daemonset.apps/kube-flannel-ds created
    ```
- Check status pods
    ```sh
    kubectl get pods --all-namespace
    ```
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
    ```sh
    kubeadm token create --print-join-command    
    ```
    # Output
    ```sh
    root@k8s-m01:~# kubeadm token create --print-join-command
    kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829 
    root@k8s-m01:~# 
    ```

<a name="3.2"></a>
### 3.2 Join Cluster worker nodes
- Worker node 1
    ```sh
    kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
    ```
    # Output
    ```sh
    root@k8s-w01:~# kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
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
    ```sh
        kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
    ```
    # Output
    ```sh
    root@k8s-w02:~# kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
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
    ```sh
    kubectl get nodes
    kubectl get nodes -o wide
    ```
    # Output
    ```sh
    root@k8s-m01:~# kubectl get nodes
    NAME      STATUS   ROLES                  AGE     VERSION
    k8s-m01   Ready    control-plane,master   31m     v1.23.4
    k8s-w01   Ready    <none>                 4m48s   v1.23.4
    k8s-w02   Ready    <none>                 98s     v1.23.4
    root@k8s-m01:~# kubectl get nodes -o wide
    NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   Ready    control-plane,master   31m     v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   containerd://1.5.5
    k8s-w01   Ready    <none>                 4m53s   v1.23.4   10.0.0.14     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   containerd://1.5.5
    k8s-w02   Ready    <none>                 103s    v1.23.4   10.0.0.15     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   containerd://1.5.5
    root@k8s-m01:~# 

    ```

<a name="4"></a>
### 4. Containerd cheat sheet command
- Plugin
    ```sh
    ctr plugins
    ctr plugins ls
    ```
    # Output
    ```sh
    root@k8s-m01:~# ctr plugins
    NAME:
    ctr plugins - provides information about containerd plugins

    USAGE:
    ctr plugins command [command options] [arguments...]

    COMMANDS:
        list, ls  lists containerd plugins

    ```
    ```sh
    root@k8s-m01:~# ctr plugins ls
    TYPE                            ID                       PLATFORMS      STATUS    
    io.containerd.content.v1        content                  -              ok        
    io.containerd.snapshotter.v1    aufs                     linux/amd64    ok        
    io.containerd.snapshotter.v1    btrfs                    linux/amd64    skip      
    io.containerd.snapshotter.v1    devmapper                linux/amd64    error     
    ---   
    io.containerd.grpc.v1           version                  -              ok        
    io.containerd.grpc.v1           cri                      linux/amd64    ok        
    root@k8s-m01:~# 

    ```
- Namespace
    ```sh
    ctr namespace
    ctr namespace ls
    ctr namespace create sinhtv
    ctr namespace rm sinhtv
    ```
    # Output
    ```sh          
    root@k8s-m01:~# ctr namespace
    NAME:
    ctr namespaces - manage namespaces

    USAGE:
    ctr namespaces command [command options] [arguments...]

    COMMANDS:
        create, c   create a new namespace
        list, ls    list namespaces
        remove, rm  remove one or more namespaces
        label       set and clear labels for a namespace

        OPTIONS:
        --help, -h  show help
        
    root@k8s-m01:~# ctr namespace ls
    NAME   LABELS 
    k8s.io        
    root@k8s-m01:~# ctr namespace create sinhtv
    root@k8s-m01:~# ctr namespace ls
    NAME   LABELS 
    k8s.io        
    sinhtv        
    root@k8s-m01:~# ctr namespace rm sinhtv
    sinhtv
    root@k8s-m01:~# ctr namespace ls
    NAME   LABELS 
    k8s.io        
    root@k8s-m01:~# 

    ```
- Images
    ```sh
    ctr image
    ctr -n k8s.io image ls
    ```
    # Output
    ```sh
    root@k8s-m01:~# 

    NAME:
        ctr images - manage images

    USAGE:
        ctr images command [command options] [arguments...]

    COMMANDS:
        check       check existing images to ensure all content is available locally
        export      export images
        import      import images
        list, ls    list images known to containerd
        mount       mount an image to a target path
        unmount     unmount the image from the target
        pull        pull an image from a remote
        push        push an image to a remote
        remove, rm  remove one or more images by reference
        tag         tag an image
        label       set and clear labels for an image
        convert     convert an image

    OPTIONS:
        --help, -h  show help  

    root@k8s-m01:~# ctr -n k8s.io image ls
    REF                                                                                                                              TYPE                                                      DIGEST                                                                  SIZE      PLATFORMS                                                                    LABELS                          
    docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1                                                                  application/vnd.docker.distribution.manifest.list.v2+json sha256:5dd61f95e28fa7ef897ff2fa402ce283e5078d334401d2f62d00a568f779f2d5 3.6 MiB   linux/amd64,linux/arm/v6,linux/arm64/v8,linux/s390x                          io.cri-containerd.image=managed 
    docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin@sha256:5dd61f95e28fa7ef897ff2fa402ce283e5078d334401d2f62d00a568f779f2d5 application/vnd.docker.distribution.manifest.list.v2+json sha256:5dd61f95e28fa7ef897ff2fa402ce283e5078d334401d2f62d00a568f779f2d5 3.6 MiB   linux/amd64,linux/arm/v6,linux/arm64/v8,linux/s390x                          io.cri-containerd.image=managed 
    ....
    sha256:5073d71511a64448fb4821c7d8a96204f16d5c5b0f6e53d1f710ae8fd883690f 14.4 MiB  linux/amd64,linux/arm/v7,linux/arm64,linux/ppc64le,linux/s390x               io.cri-containerd.image=managed 
    sha256:ed210e3e4a5bae1237f1bb44d72a05a2f1e5c6bfe7a7e73da179e2534269c459                                                          application/vnd.docker.distribution.manifest.list.v2+json sha256:1ff6c18fbef2045af6b9c16bf034cc421a29027b800e4f9b68ae9b1cb3e9ae07 294.4 KiB linux/amd64,linux/arm/v7,linux/arm64,linux/ppc64le,linux/s390x,windows/amd64 io.cri-containerd.image=managed 

    root@k8s-m01:~# 

    ```
- Containers
    ```sh
    root@k8s-m01:~# ctr container
    NAME:
    ctr containers - manage containers

    USAGE:
    ctr containers command [command options] [arguments...]

    COMMANDS:
    create           create container
    delete, del, rm  delete one or more existing containers
    info             get info about a container
    list, ls         list containers
    label            set and clear labels for a container
    checkpoint       checkpoint a container
    restore          restore a container from checkpoint

    OPTIONS:
    --help, -h  show help
        
    root@k8s-m01:~# 
    ```
- Task
    ```sh
    root@k8s-m01:~# ctr task
    NAME:
    ctr tasks - manage tasks

    USAGE:
    ctr tasks command [command options] [arguments...]

    COMMANDS:
    attach           attach to the IO of a running container
    checkpoint       checkpoint a container
    delete, rm       delete one or more tasks
    exec             execute additional processes in an existing container
    list, ls         list tasks
    kill             signal a container (default: SIGTERM)
    pause            pause an existing container
    ps               list processes for container
    resume           resume a paused container
    start            start a container that has been created
    metrics, metric  get a single data point of metrics for a task with the built-in Linux runtime

    OPTIONS:
    --help, -h  show help
        
    root@k8s-m01:~# 
    ```
