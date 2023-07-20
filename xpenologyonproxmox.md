# Встановлення DSM 7.1.1 на Proxmox 8
## Створення віртуальної машини
Для установки в напівавтоматичному режимі ми будемо використовувати Tinycore RedPill Loader від pocopico 


    # set vm id
    id=900
    
    # create image directory, download and uncomporess
    mkdir -p /var/lib/vz/images/${id}
    curl --location https://github.com/pocopico/tinycore-redpill/releases/download/v0.9.4.9/tinycore-redpill.v0.9.4.9.img.gz --output var/lib/vz/images/${id}/tinycore-redpill.img.gz
    gzip --decompress /var/lib/vz/images/${id}/tinycore-redpill.img.gz --keep

    # create disk for sata0
    pvesm alloc raid ${id} vm-${id}-disk-0.raw 30G

    # create vm
    qm create ${id} \
      --args "-drive 'if=none,id=synoboot,format=raw,file=/var/lib/vz/images/${id}/tinycore-redpill.img' -device 'qemu-xhci,addr=0x18' -device 'usb-storage,drive=synoboot,bootindex=1'" \
      --cores 2 \
      --cpu host \
      --machine q35 \
      --memory 4096 \
      --name DSM7 \
      --net0 virtio,bridge=vmbr0 \
      --numa 0 \
      --onboot 0 \
      --ostype l26 \
      --scsihw virtio-scsi-pci \
      --virtio0 raid:${id}/vm-${id}-disk-0.raw,discard=on,size=30G \
      --sockets 1 \
      --serial0 socket \
      --serial1 socket \
      --tablet 1
  
