*** Guide install Kuberneter with Docker ***  
### Menu
[1. Prepare the environment](#1)  
[2. Install environment for all nodes Kubernetes ](#2)
- [2.1 Config hostname](#2.1)
- [2.2 Install required packages ](#2.2)
- [2.3 Install Docker Engine](#2.3) 
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
    cat <<EOF >>/etc/hosts
    10.0.0.11 k8s-m01
    10.0.0.14 k8s-w01
    10.0.0.15 k8s-w02
    EOF

<a name="2.2"></a>
### 2.2 Install required packages 
    apt install -y curl wget vim nano gnupg2 software-properties-common apt-transport-https ca-certificates

<a name="2.3"></a>
### 2.3 Install Docker Engine
-  Add Docker’s official GPG key

        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
        https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
-  Install Docker Engine

        sudo apt-get update
        sudo apt-get install docker-ce docker-ce-cli containerd.io
- Setup daemon

        mkdir -p /etc/docker
        cat > /etc/docker/daemon.json <<EOF
        {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "storage-opts": [
            "overlay2.override_kernel_check=true"
        ]
        }
        EOF
        mkdir -p /etc/systemd/system/docker.service.d


- Restart containerd và enable containerd

        sudo systemctl daemon-reload
        sudo systemctl restart docker
        sudo systemctl enable docker
        sudo systemctl status docker
    # Output
    ```sh
    root@localhost:~# systemctl status containerd
    root@k8s-m01:~# systemctl status docker
    ● docker.service - Docker Application Container Engine
        Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
        Active: active (running) since Sun 2022-03-06 05:59:53 UTC; 30s ago
    TriggeredBy: ● docker.socket
        Docs: https://docs.docker.com
    Main PID: 9022 (dockerd)
        Tasks: 8
        Memory: 28.3M
        CGroup: /system.slice/docker.service
                └─9022 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.464591660Z" level=warning msg="Your kernel does not support CPU realtime scheduler"
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.464599902Z" level=warning msg="Your kernel does not support cgroup blkio weight"
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.464604109Z" level=warning msg="Your kernel does not support cgroup blkio weight_device"
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.464745792Z" level=info msg="Loading containers: start."
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.546094061Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can b>
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.580450172Z" level=info msg="Loading containers: done."
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.593642476Z" level=info msg="Docker daemon" commit=459d0df graphdriver(s)=overlay2 version=20.10.12
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.593710437Z" level=info msg="Daemon has completed initialization"
    Mar 06 05:59:53 k8s-m01 dockerd[9022]: time="2022-03-06T05:59:53.607047524Z" level=info msg="API listen on /run/docker.sock"
    Mar 06 05:59:53 k8s-m01 systemd[1]: Started Docker Application Container Engine.
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
    NAME      STATUS     ROLES                  AGE   VERSION
    k8s-m01   NotReady   control-plane,master   11m   v1.23.4
    root@k8s-m01:~# kubectl get nodes -o wide
    NAME      STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   Ready    control-plane,master   52s   v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
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
- Install a Network Plugin in the Plugin

        kubeclt apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
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

        kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
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

        kubeadm join 10.0.0.11:6443 --token 9489e3.y4jsk53zu6n3ywxs --discovery-token-ca-cert-hash sha256:0fbef9b1074945142bcdfa1852003ebce2e554db04316a77336a34821e5f0829
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

        kubectl get nodes
        kubectl get nodes -o wide
    # Output
    ```sh
    root@k8s-m01:~# kubectl get nodes
    NAME      STATUS   ROLES                  AGE     VERSION
    k8s-m01   Ready    control-plane,master   7m3s    v1.23.4
    k8s-w01   Ready    <none>                 4m33s   v1.23.4
    k8s-w02   Ready    <none>                 37s     v1.23.4
    root@k8s-m01:~# kubectl get nodes -o wide
    NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    k8s-m01   Ready    control-plane,master   7m6s    v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
    k8s-w01   Ready    <none>                 4m36s   v1.23.4   10.0.0.14     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
    k8s-w02   Ready    <none>                 40s     v1.23.4   10.0.0.15     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
    root@k8s-m01:~# 
    ```

