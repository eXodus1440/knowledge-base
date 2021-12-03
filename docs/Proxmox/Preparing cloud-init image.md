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
