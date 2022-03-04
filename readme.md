I've been able to get ceph working using a combination of https://www.digitalocean.com/community/tutorials/how-to-set-up-a-ceph-cluster-within-kubernetes-using-rook and the ceph quick start guide - I never really was happy with the idea of needing the node hosts (droplets) to have lvm2 installed, makes it hard to scale. I did how ever want to target specific nodes only using affinity rules, so here we go!

My goal is to use DO PVC option for storage vs mounted volumes

The particulars of my test cluster:
- pvc storage class name: do-block-storage
- nodes are labeled with role=storage-node which are my target nodes for ceph install (my requirement)
- RWX storage for pods 

Assumptions:
- you will want to deploy to specific nodes only
  - I do this to keep load away from pods doing other things
- you will use 3 pods for HA
- Easy to scale up or down

# Build instructions (you'll want to modify the default values to make this work)
`$ git clone --single-branch --branch v1.8.6 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml`

basically start here: https://rook.github.io/docs/rook/v1.8/quickstart.html#tldr
- `kubectl apply -f crds.yaml -f common.yaml -f operator.yaml`
(common.yaml and crds.yaml are pretty straightforward, only operator.yaml have a few changes: affinity rules and version update)
Verify your operator is running : `kubectl -n rook-ceph get pod`

Don't use the cluster.yaml because we're going to use my custom cluster-pvc.yaml example configured to run on pods with the label role=storage-node
https://github.com/webdz9r/ceph-testing/blob/master/cluster-pvc.yaml

You could optionally use the labels Digital Ocean use to identify your cluster, IE if you setup a 3 node cluster just for ceph you can the name you gave it for those affinity rules. IE: my cluster name is wbp-nodes so i'd use:
`nodeSelectorTerms:
      - matchExpressions:
        - key: doks.digitalocean.com/node-pool
          operator: In
          values:
          - wbp-premium`

Once you've configured your cluster-pvc.yaml it's type to deploy:
`k apply -f cluster-pvc.yaml` 
This generally takes 10 / 15 minutes to deploy, it has to provision 6 pods and configure the mon and osd clusters. Use what ever tool you like to watch the pods spin up in the rook-ceph namespace

Once completed you'll now see 6 new volumes on your DO account, 3 for the monitors (10gb each) and 3 for the osd filesystem cluster (100gb) all attached to your pods. 

Once your cluster is built, install the toolbox to check on ceph status and you should get something like this
`[rook@rook-ceph-tools-846884d59b-8xw7x /]$ ceph status
  cluster:
    id:     d5458d0a-b152-4ca9-95c4-a7d09045b4e9
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 8m)
    mgr: a(active, since 6m)
    osd: 3 osds: 3 up (since 5m), 3 in (since 6m)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 300 GiB / 300 GiB avail
    pgs:     1 active+clean
 `

 Which means you're ready to go

## Setting up storage class and filesystems
read more here: https://rook.github.io/docs/rook/v1.8/ceph-filesystem.html
change the values that fit your needs on shared-filesystem.yaml and shared-storageclass.yaml and deploy:
- `kubectl apply -f shared-filesystem.yaml`
- `kubectl apply -f shared-storageclass.yaml`

## Testing out your new file system
`kubectl apply -f utils/kube-registry.yaml` and verify it's using your new awesome new storage
take it down now `kubectl delete -f utils/kube-registry.yaml`



### Installing toolbox
deploy: `k apply -f utils/toolbox.yaml`
access the toolbox: `kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash`


## Notes:
There is a bug in the opperator version 1.8.4 so you need to use 
`containers:
  - name: rook-ceph-operator
    image: rook/ceph:v1.8.6`

Teardown notes: https://rook.github.io/docs/rook/v1.8/ceph-teardown.html
Doesn't apply for removing data as it's PVC

Clean up policy (might not need)
`kubectl -n rook-ceph patch cephcluster rook-ceph --type merge -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'`

Remove in order:
- `kubectl delete -f shared-storageclass.yaml -f shared-filesystem.yaml`
- `kubectl delete -f cluster-pvc.yaml`
- `kubectl delete -f crds.yaml -f common.yaml -f operator.yaml`
- `kubectl delete namespace rook-ceph`

Check if anything else is left over to manually delete
`kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n rook-ceph`