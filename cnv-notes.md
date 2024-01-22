CNV related notes
===========================================================
## Introduction
Some notes on CNV templates and related configuration

## Prepare qcow2 image using virt-install
```
sudo virt-install \
--name rhel9 \
--memory 8196 \
--vcpus 16 \
--disk size=20,bus=virtio \
--location /root/cnv/iso/RHEL-9.3.0-20231025.65-x86_64-dvd1.iso \
--os-variant rhel9.3 \
--network network=default,model=virtio \
--graphics none \
--console pty,target_type=serial \
--extra-args 'console=ttyS0,115200n8 serial'

```






