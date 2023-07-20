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
    pvesm alloc local ${id} vm-${id}-disk-0.raw 30G

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
      --sata0 local:${id}/vm-${id}-disk-0.raw,discard=on,size=30G,ssd=1 \
      --sockets 1 \
      --serial0 socket \
      --serial1 socket \
      --tablet 1
  
    ./rploader.sh update now

    ./rploader.sh fullupgrade now

    ./rploader.sh satamap now

    ./rploader.sh identifyusb now
  
    ./rploader.sh serialgen DS3622xs+



драйвер VirtIO
    ./rploader.sh ext ds3622xsp-7.1.1-42962 add https://raw.githubusercontent.com/pocopico/rp-ext/master/v9fs/rpext-index.json

драйвер Intel E1000
    ./rploader.sh ext ds3622xsp-7.1.1-42962 add https://raw.githubusercontent.com/pocopico/rp-ext/master/e1000/rpext-index.json

драйвер Realtek RTL8139
    ./rploader.sh ext ds3622xsp-7.1.1-42962 add https://raw.githubusercontent.com/pocopico/rp-ext/master/8139too/rpext-index.json



    ./rploader.sh build broadwellnk-7.1.1-42962

    ./rploader.sh build ds3622xsp-7.1.1-42962 withfriend
