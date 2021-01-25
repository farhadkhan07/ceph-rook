# Rook for Running Ceph on Kubernetes
```
# kubectl get nodes
NAME                           STATUS   ROLES                  AGE   VERSION
ceph1.yourdomain.com         Ready    <none>                 15h   v1.20.1
ceph2.yourdomain.com         Ready    <none>                 15h   v1.20.1
ceph3.yourdomain.com         Ready    <none>                 14h   v1.20.1
compute1.yourdomain.com      Ready    <none>                 14h   v1.20.1
compute2.yourdomain.com      Ready    <none>                 14h   v1.20.1
compute3.yourdomain.com      Ready    <none>                 14h   v1.20.1
controller1.yourdomain.com   Ready    <none>                 14h   v1.20.1
controller2.yourdomain.com   Ready    <none>                 14h   v1.20.1
controller3.yourdomain.com   Ready    <none>                 14h   v1.20.1
master.yourdomain.com        Ready    control-plane,master   25h   v1.20.1
```

#### If you’re feeling lucky, a simple Rook cluster can be created with the following kubectl commands. But in our case we shall do come customization to run our desired pod in specific node.
```
git clone --single-branch --branch v1.5.5 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```


#### Label the desired nodes with storage-node=true. To run Rook and Ceph daemons on labeled nodes, we will configure Rook Affinities in both the Rook Operator manifest (operator.yaml) and the Ceph cluster manifest (cluster.yaml). We shall do label our controller node here to run Rook & ceph daemons.
```
kubectl label nodes controller{1..3}.yourdomain.com role=storage-node
```
```
root@master# kubectl get node --show-labels|grep role=storage-node
controller1.yourdomain.com   Ready    <none>                 11d   v1.20.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controller1.yourdomain.com,kubernetes.io/os=linux,role=storage-node
controller2.yourdomain.com   Ready    <none>                 11d   v1.20.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controller2.yourdomain.com,kubernetes.io/os=linux,role=storage-node
controller3.yourdomain.com   Ready    <none>                 11d   v1.20.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controller3.yourdomain.com,kubernetes.io/os=linux,role=storage-node
```

#### Uncomment below line and do labeled in operator.yaml file:
```
CSI_PROVISIONER_NODE_AFFINITY: "role=storage-node"
CSI_PLUGIN_NODE_AFFINITY: "role=storage-node"
ADMISSION_CONTROLLER_NODE_AFFINITY: "role=storage-node"

- name: AGENT_NODE_AFFINITY        
          value: "role=storage-node"

- name: DISCOVER_AGENT_NODE_AFFINITY        
          value: "role=storage-node"
```
#### Uncomment below line and do node affinity in cluster.yaml file to run mon & mgr pod on desired node:
```
placement:
    mon:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: role
              operator: In
              values:
              - storage-node
    mgr:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: role
              operator: In
              values:
              - storage-node
```

### The Rook Ceph cluster property hostNetwork can be set to true to use the network of the hosts instead of using the Kubernetes network. Add the following property in cluster.yaml:
```
 network:
    # enable host networking
    provider: host

```
### This time, there are no Services created to expose the Ceph Monitor pods:
```
# kubectl get svc -n rook-ceph
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
csi-cephfsplugin-metrics   ClusterIP   10.109.125.45    <none>        8080/TCP,8081/TCP   19h
csi-rbdplugin-metrics      ClusterIP   10.97.53.77      <none>        8080/TCP,8081/TCP   19h
rook-ceph-mgr              ClusterIP   10.103.135.104   <none>        9283/TCP            19h
rook-ceph-mgr-dashboard    ClusterIP   10.104.6.176     <none>        8443/TCP            19h
```

```
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# cat /etc/ceph/ceph.conf
[global]
mon_host = 192.168.13.4:6789,192.168.13.5:6789,192.168.13.3:6789

[client.admin]
keyring = /etc/ceph/keyring
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]#
```
### In our case we have 3 osd disk for each three server. Here we are mapping the disk below.
```
  storage: # cluster level storage configuration and selection
    useAllNodes: false
    useAllDevices: false  
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
    nodes:
    - name: "ceph1.yourdomain.com"
      devices:
      - name: "vda"
      - name: "vdb"
      - name: "vdc"
    - name: "ceph2.yourdomain.com"
      devices:
      - name: "vda"
      - name: "vdb"
      - name: "vdc"
    - name: "ceph3.yourdomain.com"
      devices:
      - name: "vda"
      - name: "vdb"
      - name: "vdc"
```

```
# kubectl get pod -o wide -n rook-ceph
NAME                                                              READY   STATUS      RESTARTS   AGE   IP               NODE                           NOMINATED NODE   READINESS GATES
csi-cephfsplugin-8w25l                                            3/3     Running     0          19h   192.168.13.3     controller1.yourdomain.com   <none>           <none>
csi-cephfsplugin-gfdr4                                            3/3     Running     0          19h   192.168.13.5     controller3.yourdomain.com   <none>           <none>
csi-cephfsplugin-provisioner-557fff445-7p8nc                      6/6     Running     0          19h   10.244.199.2     controller3.yourdomain.com   <none>           <none>
csi-cephfsplugin-provisioner-557fff445-p476r                      6/6     Running     0          19h   10.244.202.133   controller2.yourdomain.com   <none>           <none>
csi-cephfsplugin-r8gq6                                            3/3     Running     0          19h   192.168.13.4     controller2.yourdomain.com   <none>           <none>
csi-rbdplugin-mzmzr                                               3/3     Running     0          19h   192.168.13.4     controller2.yourdomain.com   <none>           <none>
csi-rbdplugin-provisioner-6b67bc68d6-8xjn9                        6/6     Running     0          19h   10.244.139.129   controller1.yourdomain.com   <none>           <none>
csi-rbdplugin-provisioner-6b67bc68d6-cfh78                        6/6     Running     0          19h   10.244.199.1     controller3.yourdomain.com   <none>           <none>
csi-rbdplugin-sgkmw                                               3/3     Running     0          19h   192.168.13.3     controller1.yourdomain.com   <none>           <none>
csi-rbdplugin-vxtw7                                               3/3     Running     0          19h   192.168.13.5     controller3.yourdomain.com   <none>           <none>
rook-ceph-crashcollector-ceph1.yourdomain.com-5c6c76475dk82wd   1/1     Running     0          19h   192.168.13.9     ceph1.yourdomain.com         <none>           <none>
rook-ceph-crashcollector-ceph2.yourdomain.com-5cc97cfc6599jvn   1/1     Running     0          19h   192.168.13.10    ceph2.yourdomain.com         <none>           <none>
rook-ceph-crashcollector-ceph3.yourdomain.com-87f4cffbd-65bpq   1/1     Running     0          19h   192.168.13.11    ceph3.yourdomain.com         <none>           <none>
rook-ceph-crashcollector-controller1.yourdomain.com-7b48jgng9   1/1     Running     0          19h   192.168.13.3     controller1.yourdomain.com   <none>           <none>
rook-ceph-crashcollector-controller2.yourdomain.com-5f97zb742   1/1     Running     0          19h   192.168.13.4     controller2.yourdomain.com   <none>           <none>
rook-ceph-crashcollector-controller3.yourdomain.com-cfcbpx2j5   1/1     Running     0          19h   192.168.13.5     controller3.yourdomain.com   <none>           <none>
rook-ceph-mgr-a-5cf6d4866f-f2s6c                                  1/1     Running     0          19h   192.168.13.3     controller1.yourdomain.com   <none>           <none>
rook-ceph-mon-a-659945685c-mkhmx                                  1/1     Running     0          19h   192.168.13.3     controller1.yourdomain.com   <none>           <none>
rook-ceph-mon-b-8987d787d-hjwt2                                   1/1     Running     0          19h   192.168.13.4     controller2.yourdomain.com   <none>           <none>
rook-ceph-mon-c-56c844878c-5gl45                                  1/1     Running     0          19h   192.168.13.5     controller3.yourdomain.com   <none>           <none>
rook-ceph-operator-5798db9cb6-j574t                               1/1     Running     0          19h   10.244.103.203   compute2.yourdomain.com      <none>           <none>
rook-ceph-osd-0-69cf4d856d-fpcmd                                  1/1     Running     0          19h   192.168.13.10    ceph2.yourdomain.com         <none>           <none>
rook-ceph-osd-1-6b84c8998b-58swv                                  1/1     Running     0          19h   192.168.13.11    ceph3.yourdomain.com         <none>           <none>
rook-ceph-osd-2-56f56fb664-85ddx                                  1/1     Running     0          19h   192.168.13.9     ceph1.yourdomain.com         <none>           <none>
rook-ceph-osd-3-6d4fcc888d-7q5j6                                  1/1     Running     0          19h   192.168.13.10    ceph2.yourdomain.com         <none>           <none>
rook-ceph-osd-4-569665fc6-dvkzw                                   1/1     Running     0          19h   192.168.13.11    ceph3.yourdomain.com         <none>           <none>
rook-ceph-osd-5-7d7d54c87-7md8n                                   1/1     Running     0          19h   192.168.13.9     ceph1.yourdomain.com         <none>           <none>
rook-ceph-osd-6-74bd9b9dc8-g2sfb                                  1/1     Running     0          19h   192.168.13.10    ceph2.yourdomain.com         <none>           <none>
rook-ceph-osd-7-66d7cf9694-4vvpj                                  1/1     Running     0          19h   192.168.13.11    ceph3.yourdomain.com         <none>           <none>
rook-ceph-osd-8-6f5df8977-jmgdm                                   1/1     Running     0          19h   192.168.13.9     ceph1.yourdomain.com         <none>           <none>
rook-ceph-osd-prepare-ceph1.yourdomain.com-2458m                0/1     Completed   0          54m   192.168.13.9     ceph1.yourdomain.com         <none>           <none>
rook-ceph-osd-prepare-ceph2.yourdomain.com-wfskm                0/1     Completed   0          54m   192.168.13.10    ceph2.yourdomain.com         <none>           <none>
rook-ceph-osd-prepare-ceph3.yourdomain.com-lps8w                0/1     Completed   0          54m   192.168.13.11    ceph3.yourdomain.com         <none>           <none>
rook-ceph-tools-77bf5b9b7d-skl4t                                  1/1     Running     0          19h   10.244.202.203   compute3.yourdomain.com      <none>           <none>

```
```
# kubectl get all -n rook-ceph
NAME                                                                  READY   STATUS      RESTARTS   AGE
pod/csi-cephfsplugin-8w25l                                            3/3     Running     0          19h
pod/csi-cephfsplugin-gfdr4                                            3/3     Running     0          19h
pod/csi-cephfsplugin-provisioner-557fff445-7p8nc                      6/6     Running     0          19h
pod/csi-cephfsplugin-provisioner-557fff445-p476r                      6/6     Running     0          19h
pod/csi-cephfsplugin-r8gq6                                            3/3     Running     0          19h
pod/csi-rbdplugin-mzmzr                                               3/3     Running     0          19h
pod/csi-rbdplugin-provisioner-6b67bc68d6-8xjn9                        6/6     Running     0          19h
pod/csi-rbdplugin-provisioner-6b67bc68d6-cfh78                        6/6     Running     0          19h
pod/csi-rbdplugin-sgkmw                                               3/3     Running     0          19h
pod/csi-rbdplugin-vxtw7                                               3/3     Running     0          19h
pod/rook-ceph-crashcollector-ceph1.yourdomain.com-5c6c76475dk82wd   1/1     Running     0          19h
pod/rook-ceph-crashcollector-ceph2.yourdomain.com-5cc97cfc6599jvn   1/1     Running     0          19h
pod/rook-ceph-crashcollector-ceph3.yourdomain.com-87f4cffbd-65bpq   1/1     Running     0          19h
pod/rook-ceph-crashcollector-controller1.yourdomain.com-7b48jgng9   1/1     Running     0          19h
pod/rook-ceph-crashcollector-controller2.yourdomain.com-5f97zb742   1/1     Running     0          19h
pod/rook-ceph-crashcollector-controller3.yourdomain.com-cfcbpx2j5   1/1     Running     0          19h
pod/rook-ceph-mgr-a-5cf6d4866f-f2s6c                                  1/1     Running     0          19h
pod/rook-ceph-mon-a-659945685c-mkhmx                                  1/1     Running     0          19h
pod/rook-ceph-mon-b-8987d787d-hjwt2                                   1/1     Running     0          19h
pod/rook-ceph-mon-c-56c844878c-5gl45                                  1/1     Running     0          19h
pod/rook-ceph-operator-5798db9cb6-j574t                               1/1     Running     0          19h
pod/rook-ceph-osd-0-69cf4d856d-fpcmd                                  1/1     Running     0          19h
pod/rook-ceph-osd-1-6b84c8998b-58swv                                  1/1     Running     0          19h
pod/rook-ceph-osd-2-56f56fb664-85ddx                                  1/1     Running     0          19h
pod/rook-ceph-osd-3-6d4fcc888d-7q5j6                                  1/1     Running     0          19h
pod/rook-ceph-osd-4-569665fc6-dvkzw                                   1/1     Running     0          19h
pod/rook-ceph-osd-5-7d7d54c87-7md8n                                   1/1     Running     0          19h
pod/rook-ceph-osd-6-74bd9b9dc8-g2sfb                                  1/1     Running     0          19h
pod/rook-ceph-osd-7-66d7cf9694-4vvpj                                  1/1     Running     0          19h
pod/rook-ceph-osd-8-6f5df8977-jmgdm                                   1/1     Running     0          19h
pod/rook-ceph-osd-prepare-ceph1.yourdomain.com-2458m                0/1     Completed   0          59m
pod/rook-ceph-osd-prepare-ceph2.yourdomain.com-wfskm                0/1     Completed   0          59m
pod/rook-ceph-osd-prepare-ceph3.yourdomain.com-lps8w                0/1     Completed   0          59m
pod/rook-ceph-tools-77bf5b9b7d-skl4t                                  1/1     Running     0          19h

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/csi-cephfsplugin-metrics   ClusterIP   10.109.125.45    <none>        8080/TCP,8081/TCP   19h
service/csi-rbdplugin-metrics      ClusterIP   10.97.53.77      <none>        8080/TCP,8081/TCP   19h
service/rook-ceph-mgr              ClusterIP   10.103.135.104   <none>        9283/TCP            19h
service/rook-ceph-mgr-dashboard    ClusterIP   10.104.6.176     <none>        8443/TCP            19h

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-cephfsplugin   3         3         3       3            3           <none>          19h
daemonset.apps/csi-rbdplugin      3         3         3       3            3           <none>          19h

NAME                                                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-cephfsplugin-provisioner                            2/2     2            2           19h
deployment.apps/csi-rbdplugin-provisioner                               2/2     2            2           19h
deployment.apps/rook-ceph-crashcollector-ceph1.yourdomain.com         1/1     1            1           19h
deployment.apps/rook-ceph-crashcollector-ceph2.yourdomain.com         1/1     1            1           19h
deployment.apps/rook-ceph-crashcollector-ceph3.yourdomain.com         1/1     1            1           19h
deployment.apps/rook-ceph-crashcollector-controller1.yourdomain.com   1/1     1            1           19h
deployment.apps/rook-ceph-crashcollector-controller2.yourdomain.com   1/1     1            1           19h
deployment.apps/rook-ceph-crashcollector-controller3.yourdomain.com   1/1     1            1           19h
deployment.apps/rook-ceph-mgr-a                                         1/1     1            1           19h
deployment.apps/rook-ceph-mon-a                                         1/1     1            1           19h
deployment.apps/rook-ceph-mon-b                                         1/1     1            1           19h
deployment.apps/rook-ceph-mon-c                                         1/1     1            1           19h
deployment.apps/rook-ceph-operator                                      1/1     1            1           19h
deployment.apps/rook-ceph-osd-0                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-1                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-2                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-3                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-4                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-5                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-6                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-7                                         1/1     1            1           19h
deployment.apps/rook-ceph-osd-8                                         1/1     1            1           19h
deployment.apps/rook-ceph-tools                                         1/1     1            1           19h

NAME                                                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-cephfsplugin-provisioner-557fff445                             2         2         2       19h
replicaset.apps/csi-rbdplugin-provisioner-6b67bc68d6                               2         2         2       19h
replicaset.apps/rook-ceph-crashcollector-ceph1.yourdomain.com-5c6c76475d         1         1         1       19h
replicaset.apps/rook-ceph-crashcollector-ceph2.yourdomain.com-5cc97cfc65         1         1         1       19h
replicaset.apps/rook-ceph-crashcollector-ceph3.yourdomain.com-87f4cffbd          1         1         1       19h
replicaset.apps/rook-ceph-crashcollector-controller1.yourdomain.com-565bd7d758   0         0         0       19h
replicaset.apps/rook-ceph-crashcollector-controller1.yourdomain.com-7b489b8684   1         1         1       19h
replicaset.apps/rook-ceph-crashcollector-controller2.yourdomain.com-5f977b96df   1         1         1       19h
replicaset.apps/rook-ceph-crashcollector-controller3.yourdomain.com-cfcbbfd68    1         1         1       19h
replicaset.apps/rook-ceph-mgr-a-5cf6d4866f                                         1         1         1       19h
replicaset.apps/rook-ceph-mon-a-659945685c                                         1         1         1       19h
replicaset.apps/rook-ceph-mon-b-8987d787d                                          1         1         1       19h
replicaset.apps/rook-ceph-mon-c-56c844878c                                         1         1         1       19h
replicaset.apps/rook-ceph-operator-5798db9cb6                                      1         1         1       19h
replicaset.apps/rook-ceph-osd-0-69cf4d856d                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-1-6b84c8998b                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-2-56f56fb664                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-3-6d4fcc888d                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-4-569665fc6                                          1         1         1       19h
replicaset.apps/rook-ceph-osd-5-7d7d54c87                                          1         1         1       19h
replicaset.apps/rook-ceph-osd-6-74bd9b9dc8                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-7-66d7cf9694                                         1         1         1       19h
replicaset.apps/rook-ceph-osd-8-6f5df8977                                          1         1         1       19h
replicaset.apps/rook-ceph-tools-77bf5b9b7d                                         1         1         1       19h

NAME                                                     COMPLETIONS   DURATION   AGE
job.batch/rook-ceph-osd-prepare-ceph1.yourdomain.com   1/1           4s         59m
job.batch/rook-ceph-osd-prepare-ceph2.yourdomain.com   1/1           3s         59m
job.batch/rook-ceph-osd-prepare-ceph3.yourdomain.com   1/1           4s         59m
```


```
kubectl create -f toolbox.yaml
# kubectl get pod -n rook-ceph |grep rook-ceph-tools
rook-ceph-tools-77bf5b9b7d-skl4t                                  1/1     Running     0          19h
```
#### now login to rook-ceph-tools pod:
```
kubectl -n rook-ceph exec -it rook-ceph-tools-77bf5b9b7d-skl4t bash
```
#### From inside the pod check ceph status:
```
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# ceph status
  cluster:
    id:     67c67512-334d-4dab-8edf-f5041cbda9d7
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 19h)
    mgr: a(active, since 19h)
    osd: 9 osds: 9 up (since 19h), 9 in (since 19h)

  data:
    pools:   2 pools, 33 pgs
    objects: 0 objects, 0 B
    usage:   9.0 GiB used, 81 GiB / 90 GiB avail
    pgs:     33 active+clean

[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME                        STATUS  REWEIGHT  PRI-AFF
-1         0.08817  root default
-7         0.02939      host ceph1-yourdomain-com
 2    hdd  0.00980          osd.2                        up   1.00000  1.00000
 5    hdd  0.00980          osd.5                        up   1.00000  1.00000
 8    hdd  0.00980          osd.8                        up   1.00000  1.00000
-3         0.02939      host ceph2-yourdomain-com
 0    hdd  0.00980          osd.0                        up   1.00000  1.00000
 3    hdd  0.00980          osd.3                        up   1.00000  1.00000
 6    hdd  0.00980          osd.6                        up   1.00000  1.00000
-5         0.02939      host ceph3-yourdomain-com
 1    hdd  0.00980          osd.1                        up   1.00000  1.00000
 4    hdd  0.00980          osd.4                        up   1.00000  1.00000
 7    hdd  0.00980          osd.7                        up   1.00000  1.00000
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# ceph osd status
ID  HOST                     USED  AVAIL  WR OPS  WR DATA  RD OPS  RD DATA  STATE
 0  ceph2.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 1  ceph3.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 2  ceph1.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 3  ceph2.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 4  ceph3.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 5  ceph1.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 6  ceph2.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 7  ceph3.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
 8  ceph1.brilliant.com.bd  1027M  9208M      0        0       0        0   exists,up
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]#
```

### Storage class
```
root@master:~/rook/cluster/examples/kubernetes/ceph/csi/rbd# ls
pod.yaml  pvc-clone.yaml  pvc-restore.yaml  pvc.yaml  snapshotclass.yaml  snapshot.yaml  storageclass-ec.yaml  storageclass-test.yaml  storageclass.yaml

# kubectl create -f storageclass.yaml
```
```
# kubectl get sc
NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   18h
```
## Reference link:
```
https://rook.io/docs/rook/v1.5/ceph-quickstart.html
https://www.adaltas.com/en/2020/04/16/expose-ceph-from-rook-kubernetes/
https://documentation.suse.com/sbp/all/html/SBP-rook-ceph-kubernetes/index.html#sec-planning-nodes-ceph-daemons
https://www.youtube.com/watch?v=wIRMxl_oEMM
```
