CNV related notes
===========================================================
## Last Updated
**Last Updated:** 2024-06-18 11:30 AM

## Introduction
Some notes on CNV templates and related configuration

## Prepare qcow2 image using virt-install in rhel9
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

## windows11 VM setup

### run qcow2 image with vnc flag enabled using virt-install
```
virt-install \
  --name win11 \
  --memory 8192 \
  --vcpus 16 \
  --os-type windows \
  --os-variant win11 \
  --disk path=/root/cnv/qcow/win11.qcow2,device=disk,bus=virtio \
  --network network=default,model=virtio \
  --import \
  --graphics vnc,port=5988,listen=0.0.0.0 \
  --noautoconsole \
  --boot uefi
```

### Inject public ssh key
```
echo "xxxkeysxxx" C:\ProgramData\ssh\administrators_authorized_keys
```

### Windows batch script to get the currrent timestamp in unix timestamp

```
@echo off
setlocal

:: Get the current date and time
for /f "tokens=2 delims==" %%a in ('wmic os get localdatetime /value') do set datetime=%%a
set year=%datetime:~0,4%
set month=%datetime:~4,2%
set day=%datetime:~6,2%
set hour=%datetime:~8,2%
set minute=%datetime:~10,2%
set second=%datetime:~12,2%

:: Format the date and time
set formattedDate=%year%-%month%-%day% %hour%:%minute%:%second%

:: Calculate Unix timestamp
set /a "days=%year%*365 + %year%/4 - %year%/100 + %year%/400 + %month%*30 + %day%"
set /a "unixtime=days*86400 + hour*3600 + minute*60 + second - 719529*86400"

:: Output the timestamp and formatted date to a file
echo %unixtime%, %formattedDate% > C:\Users\Administrator\timestamp.txt
```
### Run batch script on startup
place the the batch script under this folder:
```
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```



**Last Updated:** 2024-06-18 11:30 AM
