# A draft for creating a K8S Cluster on GCP

This is just a draft for the commands I currently use to setup a K8S ckuster on GCP using kubeadm.
(I'm using Fish Shell, so some syntaxes are a bit different from normal bash).

## GCloud:

### VPC : 
	gcloud compute networks create k8s-tuto --subnet-mode custom

### Subnet:
	gcloud compute networks subnets create kubernetes \
      --network k8s-tuto \
      --range 10.240.0.0/24

### Firewalls:
	gcloud compute firewall-rules create k8s-tuto-allow-internal \
      --allow tcp,udp,icmp \
      --network k8s-tuto \
      --source-ranges 10.240.0.0/24,10.200.0.0/16

    gcloud compute firewall-rules create k8s-tuto-allow-external \
        --allow tcp:22,tcp:6443,icmp \
        --network k8s-tuto \
        --source-ranges 0.0.0.0/0

Verification:

    gcloud compute firewall-rules list --filter="network:k8s-tuto"

### Public IP Address
	gcloud compute addresses create k8s-tuto \
    --region (gcloud config get-value compute/region)

Verification:

    gcloud compute addresses list --filter="name=('k8s-tuto')"

### Kubernetes Controllers
    for i in 0
        gcloud compute instances create controller-$i \
            --async \
            --boot-disk-size 200GB \
            --can-ip-forward \
            --image-family ubuntu-1604-lts \
            --image-project ubuntu-os-cloud \
            --machine-type n1-standard-2 \
            --private-network-ip 10.240.0.1$i \
            --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
            --subnet kubernetes \
            --tags k8s-tuto,controller
    end

### Kubernetes Workers
    for i in 0
        gcloud compute instances create worker-$i \
            --async \
            --boot-disk-size 200GB \
            --can-ip-forward \
            --image-family ubuntu-1604-lts \
            --image-project ubuntu-os-cloud \
            --machine-type n1-standard-2 \
            --private-network-ip 10.240.0.2$i \
            --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
            --subnet kubernetes \
            --tags k8s-tuto,worker
    end

Verification:

    gcloud compute instances list

### On the Controller-0 Instance:
    sudo -i
    apt-get update && apt-get upgrade -y
    apt-get install -y docker.io ; \
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

    curl -s \
        https://packages.cloud.google.com/apt/doc/apt-key.gpg \
        | apt-key add - ; \
    apt-get update && apt-get install -y \
        kubeadm=1.14.1-00 kubelet=1.14.1-00 kubectl=1.14.1-00

    wget https://tinyurl.com/yb4xturm -O rbac-kdd.yaml
    wget https://tinyurl.com/y8lvqc9g -O calico.yaml

    less calico.yaml

    kubeadm init \
    --kubernetes-version 1.14.1 \
    --pod-network-cidr 192.168.0.0/16 \
    | tee kubeadm-init.out


    exit
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    kubectl apply -f rbac-kdd.yaml
    kubectl apply -f calico.yaml

### Kubectl Bash auto-complete
    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc

    sudo kubeadm token list

    openssl x509 -pubkey \
    -in /etc/kubernetes/pki/ca.crt | openssl rsa \
    -pubin -outform der 2>/dev/null | openssl dgst \
    -sha256 -hex | sed 's/^.* //'

### On the Worker-0 Instance:
    sudo -i
    apt-get update && apt-get upgrade -y
    apt-get install -y docker.io ; \
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

    curl -s \
        https://packages.cloud.google.com/apt/doc/apt-key.gpg \
        | apt-key add - ; \
    apt-get update && apt-get install -y \
        kubeadm=1.14.1-00 kubelet=1.14.1-00 kubectl=1.14.1-00

        
    kubeadm join \
    --token gd5aae.n7aeq1ygj2zh22xk \
    10.240.0.10:6443 \
    --discovery-token-ca-cert-hash \
    sha256:6aa92ec1250496d225c0c4d8a9fea9a4b351268456fd7f61d00c29a11e1447fe

### WIP....
