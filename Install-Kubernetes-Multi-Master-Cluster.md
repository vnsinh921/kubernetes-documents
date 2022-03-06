*** Guide Install Kubernetes Multi Master Cluster ***  
### Menu
[1. Prepare the environment](#1)  
[2. Install environment for all nodes Kubernetes ](#2)
- [2.1 Config hostname](#2.1)
- [2.2 Install required packages ](#2.2)
- [2.3 Install Docker Engine](#2.3) 
- [2.4 Install kubelet, kubeadm and kubectl](#2.4)

[3. Install HAProxy load balancer](#3)  
[4. Initialization Cluster](#3)
- [4.1 Install cfssl & Generating the TLS certificates (On k8s-m01)](#4.1)
- [4.2 Initialization Cluster master nodes](#4.2)
- [4.3 Joint Cluster worker nodes](#4.3)

<a name="1"></a>
### 1. Prepare the environment
    - OS: Ubuntu-20.04
    - Kubelet vesion: v1.23.4
    - Docker version: 20.10.12

    Type    Hostname	Specs               IP Addess
    HA      k8s-gw      1GB RAM, 2vcpus     IP: 10.0.0.10/24
    Master	k8s-m01     4GB Ram, 2vcpus     IP: 10.0.0.11/24
    Master	k8s-m02     4GB Ram, 2vcpus     IP: 10.0.0.12/24
    Master	k8s-m02     4GB Ram, 2vcpus     IP: 10.0.0.13/24
    Worker	k8s-w01     4GB Ram, 2vcpus     IP: 10.0.0.14/24
    Worker	k8s-w02     4GB Ram, 2vcpus     IP: 10.0.0.15/24
    Worker	k8s-w03     4GB Ram, 2vcpus     IP: 10.0.0.16/24
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


- Restart & enable docker

        sudo systemctl daemon-reload
        sudo systemctl restart docker
        sudo systemctl enable docker
        sudo systemctl status docker
    # Output
    ```sh
    root@localhost:~# systemctl status docker
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
### 3 HAProxy load balancer
- Install HAProxy

        apt-get update
        apt-get install haproxy -y
- Config HAProxy to load balance the traffic between the three Kubernetes master nodes

        vim /etc/haproxy/haproxy.cfg
        global
        ...
        default
        ...
        frontend kubernetes
        bind 10.0.0.10:6443
        option tcplog
        mode tcp
        default_backend kubernetes-master-nodes


        backend kubernetes-master-nodes
        mode tcp
        balance roundrobin
        option tcp-check
        server k8s-m01 10.0.0.11:6443 check fall 3 rise 2
        server k8s-m02 10.0.0.12:6443 check fall 3 rise 2
        server k8s-m03 10.0.0.13:6443 check fall 3 rise 2
- Restart & enable HAProxy service

        systemctl restart haproxy
        systemctl enable haproyy
        systemctl status haproxy

<a name="4"></a>
### 4 Initialization Cluster
<a name="4.1"></a>
### 4.1 Install cfssl & Generating the TLS certificates (On k8s-m01)
- Install cfssl

        wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
        wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
        chmod +x cfssl*
        mv cfssl_linux-amd64 /usr/local/bin/cfssl
        cfssl version
        mkdir - p ~/certs $$ cd ~/certs
- Create the certificate authority configuration file

        vim ca-config.json
        {
        "signing": {
            "default": {
            "expiry": "8760h"
            },
            "profiles": {
            "kubernetes": {
                "usages": ["signing", "key encipherment", "server auth", "client auth"],
                "expiry": "8760h"
            }
            }
        }
        }
- Create the certificate authority signing request configuration file

        vim ca-csr.json
        {
        "CN": "Kubernetes",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
        {
            "C": "IE",
            "L": "Cork",
            "O": "Kubernetes",
            "OU": "CA",
            "ST": "Cork Co."
        }
        ]
        }
- Generate the certificate authority certificate and private key.

        cfssl gencert -initca ca-csr.json | cfssljson -bare ca
- Creating the certificate for the Etcd cluster

        vim kubernetes-csr.json
        {
        "CN": "kubernetes",
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
        {
            "C": "IE",
            "L": "Cork",
            "O": "Kubernetes",
            "OU": "Kubernetes",
            "ST": "Cork Co."
        }
        ]
        }
- Generate the certificate and private key for the Etcd cluster

        cfssl gencert \
        -ca=ca.pem \
        -ca-key=ca-key.pem \
        -config=ca-config.json \
        -hostname=127.0.0.1,10.0.0.10,10.0.0.11,10.0.0.12,10.0.0.13,10.0.0.14,10.0.0.15,10.0.0.16,lolcalhost,k8s-gw,k8s-m01,k8s-m02,k8s-m03,k8s-w01,k8s-w02,k8s-w03,kubernetes.default \
        -profile=kubernetes kubernetes-csr.json | \
        cfssljson -bare kubernetes
- Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.

        root@k8s-m01:~/certs# ls -la
        total 44
        drwxr-xr-x  2 root root 4096 Mar  6 09:27 .
        drwx------ 11 root root 4096 Mar  6 09:31 ..
        -rw-r--r--  1 root root  232 Mar  6 09:24 ca-config.json
        -rw-r--r--  1 root root 1001 Mar  6 09:25 ca.csr
        -rw-r--r--  1 root root  194 Mar  6 09:25 ca-csr.json
        -rw-------  1 root root 1675 Mar  6 09:25 ca-key.pem
        -rw-r--r--  1 root root 1363 Mar  6 09:25 ca.pem
        -rw-r--r--  1 root root 1013 Mar  6 09:27 kubernetes.csr
        -rw-r--r--  1 root root  202 Mar  6 09:25 kubernetes-csr.json
        -rw-------  1 root root 1679 Mar  6 09:27 kubernetes-key.pem
        -rw-r--r--  1 root root 1623 Mar  6 09:27 kubernetes.pem
        root@k8s-m01:~/certs# 

- Create ETCD cluseter directory (All master nodes)

        mkdir -p /etc/kubernetes/pki/etcd 

- Copy certificate to each nodes

        scp  ~/certs/{ca-key.pem,ca.pem,kubernetes-key.pem,kubernetes.pem} root@10.0.0.11:/etc/kubernetes/pki/etcd/
        scp  ~/certs/{ca-key.pem,ca.pem,kubernetes-key.pem,kubernetes.pem} root@10.0.0.12:/etc/kubernetes/pki/etcd/
        scp  ~/certs/{ca-key.pem,ca.pem,kubernetes-key.pem,kubernetes.pem} root@10.0.0.13:/etc/kubernetes/pki/etcd/

- Rename certificate (All master nodes)

        cd /etc/kubernetes/pki/etcd/
        mv ca-key.pem cd.key
        mv ca.pem ca.crt
        mv kubernetes-key.pem server.key
        mv kubernetes.pem  server.crt

<a name="4.2"></a>
### 4.2 Initialization Cluster master nodes

- Initialization master (On node k8s-m01)

        kubeadm init --control-plane-endpoint "10.0.0.11:6443" --upload-certs --pod-network-cidr=10.244.0.0/16 
    # Output
    ```sh
        root@k8s-m01:~# kubeadm init --control-plane-endpoint "10.0.0.10:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
        [init] Using Kubernetes version: v1.23.4
        [preflight] Running pre-flight checks
        [preflight] Pulling images required for setting up a Kubernetes cluster
        [preflight] This might take a minute or two, depending on the speed of your internet connection
        [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
        [certs] Using certificateDir folder "/etc/kubernetes/pki"
        [certs] Generating "ca" certificate and key
        [certs] Generating "apiserver" certificate and key
        [certs] apiserver serving cert is signed for DNS names [k8s-m01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.11 10.0.0.10]
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
        [apiclient] All control plane components are healthy after 7.049337 seconds
        [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
        [kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
        NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
        [upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
        [upload-certs] Using certificate key:
        bf7b40e5dd0a91319176313691794ae9f787ee3f55bd1fa92cc5073e7ce0988f
        [mark-control-plane] Marking the node k8s-m01 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
        [mark-control-plane] Marking the node k8s-m01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
        [bootstrap-token] Using token: 8cbqea.hbtl19y7ku4zp43h
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

        You can now join any number of the control-plane node running the following command on each as root:

        kubeadm join 10.0.0.11:6443 --token d18bkf.0b3m8ih68az4bocz \
                --discovery-token-ca-cert-hash sha256:e1a16901c1437fd13a9bd13430da4de41f967315027b58cc8e3a267b7046bbf8 \
                --control-plane --certificate-key 69801c306e44c8924b1132904704ef60b3c13ce9cea0a7ec73727437412ce4b1

        Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
        As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
        "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

        Then you can join any number of worker nodes by running the following on each as root:

        kubeadm join 10.0.0.11:6443 --token d18bkf.0b3m8ih68az4bocz \
                --discovery-token-ca-cert-hash sha256:e1a16901c1437fd13a9bd13430da4de41f967315027b58cc8e3a267b7046bbf8 
        root@k8s-m01:~# 

    ```
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
        k8s-m01   Ready    control-plane,master   4m3s   v1.23.4
        root@k8s-m01:~# kubectl get nodes -o wide
        NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
        k8s-m01   Ready    control-plane,master   4m11s   v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12

    ```
- Check status pods

        kubectl get pods --all-namespaces
        kubectl get pods --all-namespaces -o wide
    # Output
    ```sh
        root@k8s-m01:~# kubectl get pods --all-namespaces
        NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
        kube-system   coredns-64897985d-b8gz7           0/1     Pending   0          82s
        kube-system   coredns-64897985d-hr2r7           0/1     Pending   0          82s
        kube-system   etcd-k8s-m01                      1/1     Running   1          97s
        kube-system   kube-apiserver-k8s-m01            1/1     Running   1          95s
        kube-system   kube-controller-manager-k8s-m01   1/1     Running   1          97s
        kube-system   kube-proxy-75cbk                  1/1     Running   0          82s
        kube-system   kube-scheduler-k8s-m01            1/1     Running   1          95s
    ```
- Install a Network Plugin in the Plugin

        kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
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
        NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
        kube-system   coredns-64897985d-b8gz7           1/1     Running   0          2m46s
        kube-system   coredns-64897985d-hr2r7           1/1     Running   0          2m46s
        kube-system   etcd-k8s-m01                      1/1     Running   1          3m1s
        kube-system   kube-apiserver-k8s-m01            1/1     Running   1          2m59s
        kube-system   kube-controller-manager-k8s-m01   1/1     Running   1          3m1s
        kube-system   kube-flannel-ds-w25mv             1/1     Running   0          48s
        kube-system   kube-proxy-75cbk                  1/1     Running   0          2m46s
        kube-system   kube-scheduler-k8s-m01            1/1     Running   1          2m59s
        root@k8s-m01:~# 
    ```
- Initialization master (On node k8s-m02)

        kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 \
        --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62 \
        --control-plane --certificate-key 1991f0f541e31146b072b88d8abf14fda713c71c518601c5e3cc885470b81281
    # Output:
    ```sh
        root@k8s-m02:~# kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62 --control-plane --certificate-key 1991f0f541e31146b072b88d8abf14fda713c71c518601c5e3cc885470b81281
        [preflight] Running pre-flight checks
        [preflight] Reading configuration from the cluster...
        [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
        W0306 09:44:46.223284  167742 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
        [preflight] Running pre-flight checks before initializing the new control plane instance
        [preflight] Pulling images required for setting up a Kubernetes cluster
        [preflight] This might take a minute or two, depending on the speed of your internet connection
        [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
        [download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
        [certs] Using certificateDir folder "/etc/kubernetes/pki"
        [certs] Generating "front-proxy-client" certificate and key
        [certs] Generating "etcd/peer" certificate and key
        [certs] etcd/peer serving cert is signed for DNS names [k8s-m02 localhost] and IPs [10.0.0.12 127.0.0.1 ::1]
        [certs] Generating "etcd/healthcheck-client" certificate and key
        [certs] Generating "apiserver-etcd-client" certificate and key
        [certs] Generating "etcd/server" certificate and key
        [certs] etcd/server serving cert is signed for DNS names [k8s-m02 localhost] and IPs [10.0.0.12 127.0.0.1 ::1]
        [certs] Generating "apiserver" certificate and key
        [certs] apiserver serving cert is signed for DNS names [k8s-m02 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.12 10.0.0.10]
        [certs] Generating "apiserver-kubelet-client" certificate and key
        [certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
        [certs] Using the existing "sa" key
        [kubeconfig] Generating kubeconfig files
        [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
        [kubeconfig] Writing "admin.conf" kubeconfig file
        [kubeconfig] Writing "controller-manager.conf" kubeconfig file
        [kubeconfig] Writing "scheduler.conf" kubeconfig file
        [control-plane] Using manifest folder "/etc/kubernetes/manifests"
        [control-plane] Creating static Pod manifest for "kube-apiserver"
        [control-plane] Creating static Pod manifest for "kube-controller-manager"
        [control-plane] Creating static Pod manifest for "kube-scheduler"
        [check-etcd] Checking that the etcd cluster is healthy
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Starting the kubelet
        [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
        [etcd] Announced new etcd member joining to the existing etcd cluster
        [etcd] Creating static Pod manifest for "etcd"
        [etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
        The 'update-status' phase is deprecated and will be removed in a future release. Currently it performs no operation
        [mark-control-plane] Marking the node k8s-m02 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
        [mark-control-plane] Marking the node k8s-m02 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

        This node has joined the cluster and a new control plane instance was created:

        * Certificate signing request was sent to apiserver and approval was received.
        * The Kubelet was informed of the new secure connection details.
        * Control plane (master) label and taint were applied to the new node.
        * The Kubernetes control plane instances scaled up.
        * A new etcd member was added to the local/stacked etcd cluster.

        To start administering your cluster from this node, you need to run the following as a regular user:

                mkdir -p $HOME/.kube
                sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
                sudo chown $(id -u):$(id -g) $HOME/.kube/config

        Run 'kubectl get nodes' to see this node join the cluster.

        root@k8s-m02:/etc/kubernetes/pki# 
    ```

- Initialization master (On node k8s-m03)

        kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 \
        --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62 \
        --control-plane --certificate-key 1991f0f541e31146b072b88d8abf14fda713c71c518601c5e3cc885470b81281
    # Output:
    ```sh
        root@k8s-m03:~# kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62 --control-plane --certificate-key 1991f0f541e31146b072b88d8abf14fda713c71c518601c5e3cc885470b81281
        [preflight] Running pre-flight checks
        [preflight] Reading configuration from the cluster...
        [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
        W0306 09:49:36.811855  158251 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
        [preflight] Running pre-flight checks before initializing the new control plane instance
        [preflight] Pulling images required for setting up a Kubernetes cluster
        [preflight] This might take a minute or two, depending on the speed of your internet connection
        [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
        [download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
        [certs] Using certificateDir folder "/etc/kubernetes/pki"
        [certs] Generating "front-proxy-client" certificate and key
        [certs] Generating "apiserver-etcd-client" certificate and key
        [certs] Generating "etcd/server" certificate and key
        [certs] etcd/server serving cert is signed for DNS names [k8s-m03 localhost] and IPs [10.0.0.13 127.0.0.1 ::1]
        [certs] Generating "etcd/peer" certificate and key
        [certs] etcd/peer serving cert is signed for DNS names [k8s-m03 localhost] and IPs [10.0.0.13 127.0.0.1 ::1]
        [certs] Generating "etcd/healthcheck-client" certificate and key
        [certs] Generating "apiserver" certificate and key
        [certs] apiserver serving cert is signed for DNS names [k8s-m03 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.13 10.0.0.10]
        [certs] Generating "apiserver-kubelet-client" certificate and key
        [certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
        [certs] Using the existing "sa" key
        [kubeconfig] Generating kubeconfig files
        [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
        [kubeconfig] Writing "admin.conf" kubeconfig file
        [kubeconfig] Writing "controller-manager.conf" kubeconfig file
        [kubeconfig] Writing "scheduler.conf" kubeconfig file
        [control-plane] Using manifest folder "/etc/kubernetes/manifests"
        [control-plane] Creating static Pod manifest for "kube-apiserver"
        [control-plane] Creating static Pod manifest for "kube-controller-manager"
        [control-plane] Creating static Pod manifest for "kube-scheduler"
        [check-etcd] Checking that the etcd cluster is healthy
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Starting the kubelet
        [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
        [etcd] Announced new etcd member joining to the existing etcd cluster
        [etcd] Creating static Pod manifest for "etcd"
        [etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
        The 'update-status' phase is deprecated and will be removed in a future release. Currently it performs no operation
        [mark-control-plane] Marking the node k8s-m03 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
        [mark-control-plane] Marking the node k8s-m03 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

        This node has joined the cluster and a new control plane instance was created:

        * Certificate signing request was sent to apiserver and approval was received.
        * The Kubelet was informed of the new secure connection details.
        * Control plane (master) label and taint were applied to the new node.
        * The Kubernetes control plane instances scaled up.
        * A new etcd member was added to the local/stacked etcd cluster.

        To start administering your cluster from this node, you need to run the following as a regular user:

                mkdir -p $HOME/.kube
                sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
                sudo chown $(id -u):$(id -g) $HOME/.kube/config

        Run 'kubectl get nodes' to see this node join the cluster.
        root@k8s-m03:~# 

    ```
- Check status node

        kubectl get node
        kubectl get node -o wide
    ```sh
        root@k8s-m01:~# kubectl get node
        NAME      STATUS   ROLES                  AGE     VERSION
        k8s-m01   Ready    control-plane,master   18m     v1.23.4
        k8s-m02   Ready    control-plane,master   16m     v1.23.4
        k8s-m03   Ready    control-plane,master   8m36s   v1.23.4
        root@k8s-m01:~# kubectl get node -o wide
        NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
        k8s-m01   Ready    control-plane,master   18m     v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-m02   Ready    control-plane,master   17m     v1.23.4   10.0.0.12     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-m03   Ready    control-plane,master   8m41s   v1.23.4   10.0.0.13     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        root@k8s-m01:~# 
    ```
<a name="4.3"></a>
### 4.3 Joint Cluster worker nodes
- Joint Cluster worker nodes(k8s-w01)

        kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
#Output

    ```sh
        root@k8s-w01:~# kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
        [preflight] Running pre-flight checks
        [preflight] Reading configuration from the cluster...
        [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
        W0306 10:08:08.145933   11628 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Starting the kubelet
        [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

        This node has joined the cluster:
        * Certificate signing request was sent to apiserver and a response was received.
        * The Kubelet was informed of the new secure connection details.

        Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

    ```
- Joint Cluster worker nodes(k8s-w02)

        kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
        
    ```sh

        root@k8s-w02:~# kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
        [preflight] Running pre-flight checks
        [preflight] Reading configuration from the cluster...
        [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
        W0306 10:08:08.145933   11628 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Starting the kubelet
        [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

        This node has joined the cluster:
        * Certificate signing request was sent to apiserver and a response was received.
        * The Kubelet was informed of the new secure connection details.

        Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
    ```

- Joint Cluster worker nodes(k8s-w03)

        kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
    
    ```sh
        root@k8s-w03:~# kubeadm join 10.0.0.10:6443 --token lwpozm.c36vk3jg8sic5f33 --discovery-token-ca-cert-hash sha256:d6d81cea567416f99d5051301ab68b27bf53cef727c5b5a46dc03e38a6eefa62
        [preflight] Running pre-flight checks
        [preflight] Reading configuration from the cluster...
        [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
        W0306 10:08:08.145933   11628 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
        [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
        [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
        [kubelet-start] Starting the kubelet
        [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

        This node has joined the cluster:
        * Certificate signing request was sent to apiserver and a response was received.
        * The Kubelet was informed of the new secure connection details.

        Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
    ```

- Check status node

        kubectl get node
        kubectl get node -o wide
    ```sh
        root@k8s-m01:~# kubectl get node
        NAME      STATUS   ROLES                  AGE     VERSION
        k8s-m01   Ready    control-plane,master   18m     v1.23.4
        k8s-m02   Ready    control-plane,master   16m     v1.23.4
        k8s-m03   Ready    control-plane,master   8m36s   v1.23.4
        k8s-w01   Ready    <none>                 7m20s   v1.23.4
        k8s-w02   Ready    <none>                 7m20s   v1.23.4
        k8s-w03   Ready    <none>                 7m20s   v1.23.4


        root@k8s-m01:~# kubectl get node -o wide
        NAME      STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
        k8s-m01   Ready    control-plane,master   34m     v1.23.4   10.0.0.11     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-m02   Ready    control-plane,master   33m     v1.23.4   10.0.0.12     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-m03   Ready    control-plane,master   24m     v1.23.4   10.0.0.13     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-w01   Ready    <none>                 6m25s   v1.23.4   10.0.0.14     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-w02   Ready    <none>                 6m25s   v1.23.4   10.0.0.15     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12
        k8s-w03   Ready    <none>                 6m25s   v1.23.4   10.0.0.16     <none>        Ubuntu 20.04.1 LTS   5.4.0-42-generic   docker://20.10.12

    ```
