I've been able to get ceph working using a combination of https://www.digitalocean.com/community/tutorials/how-to-set-up-a-ceph-cluster-within-kubernetes-using-rook and the ceph quick start guide

My goal is to use DO PVC option for storage vs mounted volumes

The particulars of my test cluster:
- pvc storage class name: do-block-storage
- nodes are labeled with role=storage-node which are my target nodes for ceph install (requirement)
- opperator, crds and common deploy fine

Current issues:
1. cluster fails to provision 2/3 nodes. 
I've tried to configure placements many different ways with no luck. One OSD mon will start but the opperator (see cluster-pvc and cluster-pvc-1) 3 volumes are created for the cluster but only 1 pod mounts to the volume and boots up. This puts the opperator in a crash loop for what ever reason. I have seen a multi-attach error for some of the fail cluster pods

I've attmpted to remove the affinity rules with no luck

The nodes I'm attempting to target have LVM2 installed (not sure if thats a requirement still when using PVC backed storge)

TLDR; I'm not sure if my affinity placement is wrong, not sure if DO just will not work with PVC backed storage, not sure if I'm just missing a small configuration piece


