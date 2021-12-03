# k3s on Proxmox

# Installing Proxmox
## Volume configuration
## Configuring Certs with Let's Encrypt and ACME
## Cluster creation

## SSH  keygeneration
ssh-keygen -t rsa -b 4096


# Preparing cloud-init image
https://pve.proxmox.com/wiki/Cloud-Init_Support

## download the image
wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img  # Ubuntu Server 18.04 LTS (Bionic Beaver)
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img  # Ubuntu Server 20.04 LTS (Focal Fossa)
wget https://cloud-images.ubuntu.com/hirsute/current/hirsute-server-cloudimg-amd64.img  # Ubuntu Server 21.04 (Hirsute Hippo) 

## create a new VM
qm create 9000 --memory 4096 --net0 virtio,bridge=vmbr0

## import the downloaded disk to local-lvm storage
qm importdisk 9000 focal-server-cloudimg-amd64.img local-zfs

## finally attach the new disk to the VM as scsi drive
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-zfs:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --name ubuntu-server-20.04-lts
qm template 9000

# Create k3s master nodes
qm clone 9000 1001 --name k3s-master1 --full true
qm set 1001 --ipconfig0 ip=172.16.0.41/24,gw=172.16.0.254 --nameserver 172.16.0.254 --searchdomain lan.bauer.com.au --sshkey ~/steve-rsa.pub
qm clone 9000 1002 --name k3s-master2 --full true
qm set 1002 --ipconfig0 ip=172.16.0.42/24,gw=172.16.0.254 --nameserver 172.16.0.254 --searchdomain lan.bauer.com.au --sshkey ~/steve-rsa.pub

# Create k3s worker nodes
qm clone 9000 1011 --name k3s-worker1 --full true
qm set 1011 --ipconfig0 ip=172.16.0.31/24,gw=172.16.0.254 --nameserver 172.16.0.254 --searchdomain lan.bauer.com.au --sshkey ~/steve-rsa.pub
qm clone 9000 1012 --name k3s-worker2 --full true
qm set 1012 --ipconfig0 ip=172.16.0.32/24,gw=172.16.0.254 --nameserver 172.16.0.254 --searchdomain lan.bauer.com.au --sshkey ~/steve-rsa.pub
qm clone 9000 1013 --name k3s-worker3 --full true
qm set 1013 --ipconfig0 ip=172.16.0.33/24,gw=172.16.0.254 --nameserver 172.16.0.254 --searchdomain lan.bauer.com.au --sshkey ~/steve-rsa.pub


# Cloud-Init
## Installing management tools
- https://brew.sh/
- https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/


## Installing k3s/k3os
- https://github.com/k3s-io/k3s-ansible

ansible-playbook site.yml -i inventory/k3s-cluster/hosts.ini --private-key ~/Documents/_keys/steve_rsa


## Installing Rancher dashboard
- https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/

GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
sudo kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml

sudo kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
sudo kubectl -n kubernetes-dashboard describe secret admin-user-token | grep '^token'
sudo kubectl proxy

## Installing ArgoCD
- https://argo-cd.readthedocs.io/en/stable/getting_started/
sudo kubectl create namespace argocd
sudo kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
## Deleting ArgoCD
sudo kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## Installing ArgoCD via Helm
- https://github.com/HoussemDellai/argo-cd-demo
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade argocd argo/argo-cd --namespace argo-system --install --create-namespace

## Installing ArgoCD Demo
git clone https://gitlab.com/nanuchi/argocd-app-config.git
sudo kubectl apply -n argocd -f https://gitlab.com/nanuchi/argocd-app-config/-/raw/main/application.yaml
