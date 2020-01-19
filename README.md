# microk8s-multinode-nfs
Pourpose of this repo is to show basic setup and usage of MicroK8s.

# VM setup

## VM disk image
In this tutorial I will use VM setup. The downloaded image was Ubuntu 18.04. It was run under VMWare Player 15.
https://www.osboxes.org/ubuntu/#ubuntu-1804-vmware

You will need to make two copies of disk image and create two virtual machines.

## Required VM configuration
For convinience you can disable automatic updates on VM.
https://linuxconfig.org/disable-automatic-updates-on-ubuntu-18-04-bionic-beaver-linux

**Important:** MicroK8s require different hostnames for each node. You **have to** change name of one VM.
https://www.cyberciti.biz/faq/ubuntu-change-hostname-command/

You will need to install required packages on both VMs.
```
sudo apt-get install snapd
sudo snap install core
sudo apt-get install nfs-common
```
**Important:** nfs-common packet is required for every node. Without it kubernetes pod is not able to mount nfs volume.

However net tools are not required, but I suggest to install it for route command.
```
sudo apt-get install net-tools
```

# MicroK8s setup

## Instalation
This section is based on MicroK8s Quick start page: https://microk8s.io/docs/
```
sudo snap install microk8s --classic --channel=1.16/stable
sudo usermod -a -G microk8s $USER
su - $USER
sudo iptables -P FORWARD ACCEPT
```
Above commands need to be run on both VMs.
Now you need to choose which VM will serve as master for MicroK8s and which will serve only as node. (By default master is also node and will run pods).

_Note:_ change in iptables is required to forward ports between pods.

## Master configuration
On master node you will use microk8s.kubectl to menage cluster. You will need to change default configuration.

**Important:** nfs server pod which will be run in next part of tutorial require privileged access. You need to explicite enable this by writing `--allow-privileged=true` to the following file:
```
sudo vim /var/snap/microk8s/current/args/kube-apiserver
```
If you change config files you should restart MicroK8s. It will be covered in section **Useful commands**.

Now you can start MicroK8s:
```
microk8s.start
```
Next part cover conecting nodes into cluster. It is based on https://microk8s.io/docs/clustering.
On master run:
```
microk8s.add-node
```
You will get output similar to this:
```
Join node with: microk8s.join 172.16.209.130:25000/wBKwiPCoZGDqwfnjvfdemJXxYgyeJYGv

If the node you are adding is not reachable through the default interface you can use one of the following:
 microk8s.join 172.16.209.130:25000/wBKwiPCoZGDqwfnjvfdemJXxYgyeJYGv
 microk8s.join 172.17.0.1:25000/wBKwiPCoZGDqwfnjvfdemJXxYgyeJYGv
 microk8s.join 10.1.7.0:25000/wBKwiPCoZGDqwfnjvfdemJXxYgyeJYGv
```
## Node configuration
On second node which is not master you need to run join command proposed by add-node. You will need to choose ip address which is accessable from node (check with ping).
```
microk8s.join 172.16.209.130:25000/wBKwiPCoZGDqwfnjvfdemJXxYgyeJYGv
```
_Note_: token in join command can be used only once. You will need to generete new token on master to add different node.

To check if node is added correctly you can run on master:
```
$ microk8s.kubectl get no -o wide
NAME             STATUS   ROLES    AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
172.16.209.131   Ready    <none>   8m8s   v1.16.4   172.16.209.131   <none>        Ubuntu 18.04.3 LTS   5.0.0-23-generic   containerd://1.2.5
osboxes          Ready    <none>   12m    v1.16.4   172.16.209.130   <none>        Ubuntu 18.04.3 LTS   5.0.0-23-generic   containerd://1.2.5
```
_Note_: INTERNAL-IP of master match with ip used in join command.

# MicroK8s NFS ping pong
In this part we will setup nfs server and two workers on cluster. Config files are present in this repository **TODO**. They are based on following tutorial. You can read it to learn something more.
https://github.com/kubernetes/examples/tree/master/staging/volumes/nfs

If you prefer to check status of cluster (nodes, pods, pv, pvc, services, etc.) in browser then in CLI, you can go first to **MicroK8s dashboard** section now and return here later.

## Setup NFS server
First you need to create service. It will describe which network ports are open. It will also create service ip address which will be used to comunicate with pod which implement this service.
```
microk8s.kubectl apply -f nfs-server-service.yaml
```
After successful creating nfs server service, you will need to check its ip address.
```
$ microk8s.kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                              AGE     SELECTOR
kubernetes   ClusterIP   10.152.183.1    <none>        443/TCP                              8m12s   <none>
nfs-server   ClusterIP   10.152.183.38   <none>        2049/TCP,20048/TCP,111/TCP,111/UDP   29s     role=nfs-server
```
You will need to change nfs-pv.yaml file. It need to points to nfs-server service ip address.
```
...
  nfs:
    #       VVVVVVVVVVVVV -- service ip go there
    server: 10.152.183.38
    path: "/"
```
After that you can create persistent volume and persistent volume claim. The first one describe volume parameters. The second one is used by pods in their description.
```
microk8s.kubectl apply -f nfs-pv.yaml
microk8s.kubectl apply -f nfs-pvc.yaml
```
_Note_: pv and pvc applied on kubernetes can be shown using kubectl get with pv or pvc argument. It is similar to geting services (svc) and nodes (no).

Now you can run nfs server. We will use replication controler. This controler makes sure that there is specified number of pods with described configuration. We will create one nfs server.
```
microk8s.kubectl apply -f nfs-server-rc.yaml
```
You can see that replication controler is created.
```
$ microk8s.kubectl get rc
NAME          DESIRED   CURRENT   READY   AGE
nfs-server    1         1         1       49s
```
You will see also created pod. You can think about pods as separate docker containers. They are created from docker image, has own ip address and are independent (but may require some host tools e.g. nfs-common for mounting nfs).
```
$ microk8s.kubectl get pod -o wide
NAME                READY   STATUS              RESTARTS   AGE   IP       NODE             NOMINATED NODE   READINESS GATES
nfs-server-nv5kl    0/1     ContainerCreating   0          13s   <none>   osboxes          <none>           <none>
```
You can see that in my case nfs server was deployed on my master node. After status change to Running it should be possible to mount this share using service ip or pod ip on node. Here is example of mounting using service ip.
```
$ sudo mount -t nfs 10.152.183.38:/exports ~/tmpmnt -v
[sudo] password for osboxes: 
mount.nfs: timeout set for Sun Jan 19 12:27:43 2020
mount.nfs: trying text-based options 'vers=4.2,addr=10.152.183.38,clientaddr=172.16.209.131'
mount.nfs: mount(2): No such file or directory
mount.nfs: trying text-based options 'addr=10.152.183.38'
mount.nfs: prog 100003, trying vers=3, prot=6
mount.nfs: trying 10.152.183.38 prog 100003 vers 3 prot TCP port 2049
mount.nfs: prog 100005, trying vers=3, prot=17
mount.nfs: trying 10.152.183.38 prog 100005 vers 3 prot UDP port 20048
mount.nfs: portmap query retrying: RPC: Timed out
mount.nfs: prog 100005, trying vers=3, prot=6
mount.nfs: trying 10.152.183.38 prog 100005 vers 3 prot TCP port 20048
```
And here is example of mounting using pod id. First previous mount is unmounted.
```
$ sudo umount ~/tmpmnt
$ sudo mount -t nfs 10.1.7.2:/exports ~/tmpmnt -v
mount.nfs: timeout set for Sun Jan 19 12:29:00 2020
mount.nfs: trying text-based options 'vers=4.2,addr=10.1.7.2,clientaddr=10.1.88.0'
mount.nfs: mount(2): No such file or directory
mount.nfs: trying text-based options 'addr=10.1.7.2'
mount.nfs: prog 100003, trying vers=3, prot=6
mount.nfs: trying 10.1.7.2 prog 100003 vers 3 prot TCP port 2049
mount.nfs: prog 100005, trying vers=3, prot=17
mount.nfs: trying 10.1.7.2 prog 100005 vers 3 prot UDP port 20048
```
_Note_: service ip is static and this one was used in PV configuration. Pod ip can change every time pod for this service is created.
_Note_: nfs server pod is using its temporary working volume as storage. It will be different for each nfs server pod. You can mount different volume to nfs server and then share this mount using nfs. It will be similar configuration to that in nfs-pv, nfs-pvc and nfs-busybox-rc scenario. This setup is beyond the scope of this tutorial.

## Setup workers
Now we will setup pods which will use storage exported by nfs server. They will be write to the same file. Run on master:
```
microk8s.kubectl apply -f nfs-busybox-rc.yaml
```
You can check that pod were started.
```
$ microk8s.kubectl get pod -o wide
NAME                READY   STATUS    RESTARTS   AGE     IP          NODE             NOMINATED NODE   READINESS GATES
nfs-busybox-8qn7z   1/1     Running   0          2m12s   10.1.88.2   172.16.209.131   <none>           <none>
nfs-busybox-hbdkd   1/1     Running   0          2m12s   10.1.7.3    osboxes          <none>           <none>
nfs-server-nv5kl    1/1     Running   0          2m12s   10.1.7.2    osboxes          <none>           <none>
```
In my case one worker pod started on master and one on other node. You will see that pods on the same node are in the same subnet. In default configuration all pods has address from 10.1.0.0/16. Each node has its own subnet. In my case master has 10.1.7.0/24 and second node has 10.1.88.0/24. Thanks to that pods are in the same LAN. However with MicroK8s there is supplied flannel tool. It manages routing between nodes.
```
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.209.2    0.0.0.0         UG    100    0        0 ens33
10.1.7.0        0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.1.88.0       10.1.88.0       255.255.255.0   UG    0      0        0 flannel.1
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 ens33
172.16.209.0    0.0.0.0         255.255.255.0   U     100    0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```
You can see that 10.1.7.0 subnet is directly connected to master, while 10.1.88.0 is routed by flannel. Both cni0 and flannel.1 are logic interfaces.

The script that is running on workers is written in nfs-busybox-rc.yaml. They should change content of the index.html file on nfs server. You can check it mounting nfs and displaing index.html file.
```
$ cat ~/tmpmnt/index.html 
Sun Jan 19 15:35:27 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 15:35:28 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 15:35:34 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 15:35:36 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 15:35:40 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 15:35:44 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 15:35:46 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 15:35:53 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 15:35:54 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 15:35:59 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 15:35:59 UTC 2020
nfs-busybox-hbdkd
```
You can also access pod shell directly from master usin pod's name:
```
$ microk8s.kubectl exec -it nfs-busybox-8qn7z -- sh
/ # mount | grep nfs
10.152.183.38:/ on /mnt type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.16.209.131,local_lock=none,addr=10.152.183.38)
/ # ls mnt
index.html
/ # cat mnt/index.html
Sun Jan 19 17:52:13 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:15 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 17:52:20 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:24 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 17:52:26 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:31 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:32 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 17:52:39 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:40 UTC 2020
nfs-busybox-hbdkd
Sun Jan 19 17:52:45 UTC 2020
nfs-busybox-8qn7z
Sun Jan 19 17:52:49 UTC 2020
nfs-busybox-hbdkd
/ # 
```

# MicroK8s dashboard
In MicroK8s you can easily configure dashboard which allows you to view cluster status from browser. It is described in https://microk8s.io/docs/addon-dashboard
Run on master:
```
microk8s.enable dashboard
token=$(microk8s.kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s.kubectl -n kube-system describe secret $token
```
The generated token will look like this:
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IlByWDJHbnNYTFhIenBRVGRCM2hadFFmOWNLaTU5Y01LRmpUUjFFbVVNUncifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iisia3ViZXJuZYRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZXZhdWx5LXRva2VuLWpsa2c3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmY2MyYzIzYy0xOTU5LTQwNGYtOWI2NS04ZGE2MDBlMjBmOWQiLCJzdWIiOiJzeXN0ZW06c2VydmljAWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.r_UQAKsUWO3AXynQn_9VeeiwQ3c8dHOC-Bai7Bvd1tk0oXehKzwQ4y0w6LY2JpyreQ9Msq7t8lubfgttJ0yIbn5f0h3NDCDr2zb2Ze1gOShqR70BKWbiGhiGknBAhqfMdzhKB0ej7wrD9YrmbjK0PbGlop67HoKMelPBeTABczPNNGdqD8pHFsJkKEs2LP8ggWNJqewdKfnO61XRTxRI8_f9WcvCSPIy07Mz8HDFyviMYj43YhwixmfOJ1hz8GBieC7lTqwqjs5y6KHvOLtsHvUXFoOjQA2hU_HKnJb_8Qxoz_eJYVRBF-o0ElhrEcY8tBBXymsph57UsA723QqWDg
```
You will use it to login to the dashboard.

You can skip --address option in next command, but without making dashboard public you will be not able to connect it from your native host.
```
microk8s.kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443 --address 0.0.0.0
```
Dashboard is available on localhost (when you are using browser in your master VM) or on address specified in option (if address is 0.0.0.0 then dashobard is available through all network interfaces). Use following address in your browser (change ip if required).
```
https://172.16.209.130:10443
```
_Note_: https protocol is used for comunication. You may need to add certificate exception in your browser.

# Useful commands
After changing configuration files you should use following sequence to reset MicroK8s:
```
microk8s.stop
microk8s.start
```
If you want to clean all configuration of kubernetes (created pods, services, replicators, deployments, etc.) you can use:
```
microk8s.reset
```
If you need to restore default MicroK8s configuration, you can remove it. Later you will need to download it again.
```
sudo snap remove microk8s
```
In CLI all objects (pods, pvc, pv, nodes, services, etc.) can be chacked using get command. With option `-o wide` this command gives more details.
```
microk8s.kubectl get pod -o wide
```
Each object can be removed using command delete. Its syntax is similar to get:
```
microk8s.kubectl delete pod nfs-busybox-c255b
```
By adding `--grace-period=0 --force` you can force delate (sometimes normal is hanging and need to be break by control+C. microk8s.reset may also hang on deleting. You can look for item to delete and use force).
```
microk8s.kubectl delete pvc nfs --grace-period=0 --force
```
Apply may update configuration of kubernetes objects as well as creating theme. Pod that are already running are not updated.
```
microk8s.kubectl apply -f nfs-server-rc.yaml
```
PV cannot be updated. First you have to delete coressponding PVC and later delete PV and recreate it.
```
microk8s.kubectl delete pvc nfs
microk8s.kubectl delete pv nfs
microk8s.kubectl apply -f nfs-pv.yaml
```
You can run commands on pods. Including interactive shell.
```
microk8s.kubectl exec -it nfs-server-8qn7z -- sh
```
Deleting replication controler will stop all coresponding pods.
```
microk8s.kubectl delete rc nfs-busybox
```
