Understanding the difference pvc/snapshot clone in ceph 
===========================================================
## Last Updated
**Last Updated:** 2024-06-20 15:03 PM

## Introduction
ceph has been a black box from the view point of CNV and cloning a volume via pvc can be quite different compared to cloning from a snapshot. I took some time by looking into what's happening within the ceph backend when PVC/snapshot cloning happens and highlight some of the interesting and important differences that are relavant to CNV VM performance and scale. 

### PVC clone
```
source:
  pvc:
    name: rhel9-parent
```

### snapshot clone
```
 source:
   snapshot:
     name: rhel9-snapshot
```

## PVC Clone
I have created a data volume object which imports the qcow2 image of a VM. We can later use it as the (grand)parent image of all other VM clones.

```
oc get dv | grep parent
rhel9-parent   Succeeded   100.0%     1          6d5h
```

```
oc get pvc | grep parent
rhel9-parent   Bound    pvc-dddbe126-bf25-4441-a418-effa3a2bf794   21Gi       RWX            ocs-storagecluster-ceph-rbd-virtualization   6d5h
```

When describing the dv object, we notice that the source is coming from a URL source where the qcow2 image is served by a Python HTTP server I am running on another machine.
```
  source:                                         
    http:                                         
      url: http://10.16.29.214:8080/rhel9.4.qcow2
```

Then we can create a VM by cloning from this pvc. 

```
oc get dv
root-1         Succeeded   100.0%                83m
```
describing root-1, we can see that, this root disk is cloning directly from rhel9-parent pvc.
```
    Resources:
      Requests:
        Storage:         21Gi
    Storage Class Name:  ocs-storagecluster-ceph-rbd-virtualization
    Volume Mode:         Block
  Source:
    Pvc:
      Name:       rhel9-parent
      Namespace:  default
```
We get into the ceph backend, we see that there are already 6 snapshots created for os images and one rdb volume for db-noobaa database. We can simply ignore those volumes.
```
sh-5.1$ rbd ls ocs-storagecluster-cephblockpool         
csi-snap-056e20e6-2bd8-4f85-aaed-e60febdfe6ed
csi-snap-0b042194-f980-42f6-80e3-235dd837509c
csi-snap-28a25c3b-70fa-4624-bac2-8e9e478f44a1
csi-snap-2cc688eb-111b-4b72-b65c-05a7b480cb2c
csi-snap-b07c0baa-0cf8-4f43-aa57-8501e5436f79
csi-snap-cdee74a1-d0d7-4949-b65e-d2bfd7c911e1
csi-vol-3e051c30-9ece-4f15-bf70-ac4a05abc201
```

To figure out the image id of all those volumes, we can go to the ceph backend and poke around and see how they are actually residing in ceph cluster. The image id is stored in persistent volume object, by describing those PVs,  we can get the image IDs and its corresponding volumes:
```
rhel9-parent: csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2
root-1: csi-vol-73f44909-a673-4300-bf02-97c044b51fa0
```
The interesting thing is that, whenever we do a clone from PVC, two volumes will be created, one is appended with temp suffix and another one without it.
```
rbd ls ocs-storagecluster-cephblockpool | grep csi-vol-73f44909-a673-4300-bf02-97c044b51fa0
csi-vol-73f44909-a673-4300-bf02-97c044b51fa0
csi-vol-73f44909-a673-4300-bf02-97c044b51fa0-temp
```
Examining the image one by one, first start with the image we imported from the URL source. No parent tag which is what we should expect since it's imported directly from the URL source, not clones.
```
rbd info ocs-storagecluster-cephblockpool/csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2
rbd image 'csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2':
        size 21 GiB in 5376 objects
        order 22 (4 MiB objects)
        snapshot_count: 1
        id: 41f040cd3e15
        block_name_prefix: rbd_data.41f040cd3e15
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, operations
        op_features: clone-parent, snap-trash
        flags: 
        create_timestamp: Sat Jun  8 01:43:53 2024
        access_timestamp: Sat Jun  8 01:43:53 2024
        modify_timestamp: Sat Jun  8 01:43:53 2024
```
And for the root-1's image, it doesn't have a tmp suffix, and we notice that it has a parent tag and its parent is the volume with temp suffix.
```
rbd info ocs-storagecluster-cephblockpool/csi-vol-73f44909-a673-4300-bf02-97c044b51fa0
rbd image 'csi-vol-73f44909-a673-4300-bf02-97c044b51fa0':
        size 21 GiB in 5376 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 14f13bda474c8b
        block_name_prefix: rbd_data.14f13bda474c8b
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, operations
        op_features: clone-child
        flags: 
        create_timestamp: Mon Jun 17 07:35:04 2024
        access_timestamp: Mon Jun 17 07:35:04 2024
        modify_timestamp: Mon Jun 17 07:35:04 2024
        parent: ocs-storagecluster-cephblockpool/csi-vol-73f44909-a673-4300-bf02-97c044b51fa0-temp@b142fe70-b793-4671-bfed-cd8bee998c87
        overlap: 21 GiB
```

if we take a look at the temp image, it also has a parent image which is the base image rhel9-parent imported directly from URL source.
```
rbd info ocs-storagecluster-cephblockpool/csi-vol-73f44909-a673-4300-bf02-97c044b51fa0-temp
rbd image 'csi-vol-73f44909-a673-4300-bf02-97c044b51fa0-temp':
        size 21 GiB in 5376 objects
        order 22 (4 MiB objects)
        snapshot_count: 1
        id: 14f13b4aa27389
        block_name_prefix: rbd_data.14f13b4aa27389
        format: 2
        features: layering, deep-flatten, operations
        op_features: clone-parent, clone-child, snap-trash
        flags: 
        create_timestamp: Mon Jun 17 07:35:03 2024
        access_timestamp: Mon Jun 17 07:35:03 2024
        modify_timestamp: Mon Jun 17 07:35:03 2024
        parent: ocs-storagecluster-cephblockpool/csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2@cca6b09b-4808-468c-9df0-cd2f0ad2a64d
        overlap: 21 GiB
```

So what ceph did here was to create a temp vlomue clone of the rhel9-parent image and then use that as a parent of the actual vm vloume root-1. The relationship can be describe as : grandparent(rhel9-parent) -> parent(xx-temp) -> child (xx). 

The actual size of each image is identical which make mes think they are the full clone of the original image, but then later when I take a look at the space ultilization, it made me realized that they are just referencing to the same original image.
```
sh-5.1$ rbd diff ocs-storagecluster-cephblockpool/csi-vol-73f44909-a673-4300-bf02-97c044b51fa0 | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'                                                                                                       
4377.25 MB

sh-5.1$ rbd diff ocs-storagecluster-cephblockpool/csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2 | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
4377.25 MB

sh-5.1$ rbd diff ocs-storagecluster-cephblockpool/csi-vol-73f44909-a673-4300-bf02-97c044b51fa0-temp | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'                                                                                            
4377.25 MB
```
So when we do a PVC clone, there will be two volumes created: one with temp suffix and one without, this seem to workaround the limit of snapshots since the subsquent snapshots will be created off this unique temp volume.

## Snapshot Clone
A different method was later introduced to clone from a snapshot like this:
```
source:
  snapshot:
    namespace: default
    name: rhel9-snap
```
We need to first create a snapshot of this rhel9-parent image. On the ceph backend, a snapshot image is named like this, with csi-snap as the prefix:
```
csi-snap-a16525c8-8556-4a88-bca4-f6d542bc6cbf
```
By looking at the metadata of this snapshot image, we see that its parent is indeeded the rhel9-parent image.
```
 rbd info  ocs-storagecluster-cephblockpool/csi-snap-a16525c8-8556-4a88-bca4-f6d542bc6cbf
rbd image 'csi-snap-a16525c8-8556-4a88-bca4-f6d542bc6cbf':
        size 21 GiB in 5376 objects
        order 22 (4 MiB objects)
        snapshot_count: 1
        id: cee6655654cd1
        block_name_prefix: rbd_data.cee6655654cd1
        format: 2
        features: layering, deep-flatten, operations
        op_features: clone-parent, clone-child
        flags: 
        create_timestamp: Mon Jun 17 12:20:34 2024
        access_timestamp: Mon Jun 17 12:20:34 2024
        modify_timestamp: Mon Jun 17 12:20:34 2024
        parent: ocs-storagecluster-cephblockpool/csi-vol-6ea3447e-0e75-4dd3-9cba-5fee81a7b0b2@231e26c7-eff9-4f56-879c-bb3503e03f8e
        overlap: 21 GiB
```
Also there is no temp volume anymore, but just a child volume of this snapshot:
```
rbd info  ocs-storagecluster-cephblockpool/csi-vol-b589183d-78c6-4379-aa7d-f416adfffe66
rbd image 'csi-vol-b589183d-78c6-4379-aa7d-f416adfffe66':
        size 21 GiB in 5376 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 14f13b979213bd
        block_name_prefix: rbd_data.14f13b979213bd
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, operations
        op_features: clone-child
        flags: 
        create_timestamp: Mon Jun 17 12:28:21 2024
        access_timestamp: Mon Jun 17 12:28:21 2024
        modify_timestamp: Mon Jun 17 12:28:21 2024
        parent: ocs-storagecluster-cephblockpool/csi-snap-a16525c8-8556-4a88-bca4-f6d542bc6cbf@csi-snap-a16525c8-8556-4a88-bca4-f6d542bc6cbf
        overlap: 21 GiB
```
For every subsequent volume clone, it will all point back to the same snapshot image. 

## Storage Space
### PVC clone
Although the terminology used is "volume clone", but when I looked at the disk space usage before and after creating 100 VMs, the difference is only marginal.

Before any VM creation
```
ceph df | grep ocs-storagecluster-cephblockpool                                                                       
ocs-storagecluster-cephblockpool                        2  256   19 GiB    5.27k   56 GiB   0.09     21 TiB
```

After 100 VMs creation
```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool 
ocs-storagecluster-cephblockpool                        2  256   19 GiB    5.90k   56 GiB   0.09     21 TiB
```
We see that the total number of objects increased from 5.27k to 5.90k, However, the total storage ultilization is not changed, this is probably because ceph was just creating references point back to the parent image. Since there is a soft limit of 250 clones before flattening happens, let's try cloning > 250 images and monitor how the storage ultilization of that particular pool changes. When the number clones is at 252, we see the storage ultilization increase by 16GiB which indicates falttening process indeed happened to some degree. 

```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool
ocs-storagecluster-cephblockpool                        2  256   35 GiB   11.18k   81 GiB   0.12     21 TiB
``` 
The interesting is that, when I looped through all the rbd images, including both the temp and non-temp volumes, parent tag still exists, so what's been flattened here is still not so clear, since the parent link is still there.

### Snapshot clone
Let's also take a look at how snapshot cloning different from volume cloning in terms of storage space ultilization. I have created 249 snapshot clones, the storage space is the same as before:
```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool
ocs-storagecluster-cephblockpool                        2  256   19 GiB    6.20k   56 GiB   0.09     21 TiB
```
Then I created 496 snapshot clones, the storage ultilization seems to stays the same which makes me think the soft limit 250 is not applied to snapshot clone.
```
 ceph df | grep ocs-storagecluster-cephblockpool
ocs-storagecluster-cephblockpool                        2  256   19 GiB    6.63k   56 GiB   0.09     21 TiB

``
Let's try a snapshot clone of 512, the storage ultilization is still the same, cloning process was fairly quickly, this makes sense since it was just copying references:
```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool 
ocs-storagecluster-cephblockpool                        2  256   19 GiB    7.25k   56 GiB   0.09     21 TiB
```

With snapshot clone of 515, storage ultilization stays the same:
```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool 
ocs-storagecluster-cephblockpool                        2  256   19 GiB    7.26k   56 GiB   0.09     21 TiB
```
With snapshot clone of 1024, storage ultilization stays the same, looks like the the limit for snapshot cloning is set at much higher bar, at least we haven't hit the soft limit at 1024 snapshots clones.
```
sh-5.1$ ceph df | grep ocs-storagecluster-cephblockpool 
ocs-storagecluster-cephblockpool                        2  256   19 GiB    9.30k   56 GiB   0.09     21 TiB
```


