About
=====

Trying to run babel routing daemon inside a docker container.

Why?
====

* Run many babeld on multiple containers
* Load testing
* Simulations
* Generate packet loss and latency via netem module  
* Etc...

Status
======

WIP, I can't make 2 containers to see each other, don't know why...

Pre-built image
===============

If you want to try, here is a oneliner:

```
$ docker run --privileged zoobab/babeld-in-docker
Warning: couldn't find router id -- using random value.
Type: 0
Noticed ifindex change for eth0.
Noticed status change for eth0.
Sending seqno 49393 from address 0x61b070 (dump)
Netlink message: [multi] {seq:49393}(msg -> "found address on interface lo(1): 127.0.0.1
" 0), [multi] {seq:49393}(msg -> "found address on interface eth0(44): 172.17.0.2
[...]
```

Docker with IPv6
================

You need to configure Docker to use IPv6, which is not by default:

https://docs.docker.com/engine/userguide/networking/default_network/ipv6/

In my case, it means adding this config file "/etc/docker/daemon.json":

```
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
```

And then restart the docker daemon with:

```
$ systemctl restart docker
```

Then start a container and check that eth0 has an IPv6 address:

```
$ docker run -it alpine:3.6 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:06
          inet addr:172.17.0.6  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:6/64 Scope:Link
          inet6 addr: 2001:db8:1::242:ac11:6/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:594 (594.0 B)  TX bytes:266 (266.0 B)
```

Run babel inside docker
=======================

By default we use a LEDE image to build a rootfs with babeld 1.8.0 inside (see the Dockerfile).

If you want to build it, do:

```
$ docker build -t babeld .
```

This will produce an image called "babeld:latest":

```
$ docker images | grep babeld
babeld    	latest              10f14f275ae2        14 minutes ago      8.13MB
```

Now run the image:

```
$ docker run --privileged babeld
Warning: couldn't find router id -- using random value.
Noticed ifindex change for eth0.
Noticed status change for eth0.
Type: 0
Sending seqno 48915 from address 0x61b070 (dump)
Netlink message: [multi] {seq:48915}(msg -> "found address on interface lo(1): 127.0.0.1
" 0), [multi] {seq:48915}(msg -> "found address on interface eth0(42): 172.17.0.2
" 0),
Netlink message: [multi] {seq:48915}(msg -> "found address on interface lo(1): ::1
" 0), [multi] {seq:48915}(msg -> "found address on interface eth0(42): 2001:db8:1::242:ac11:2
" 0), [multi] {seq:48915}(msg -> "found address on interface eth0(42): fe80::42:acff:fe11:2
" 1),
Netlink message: [multi] {seq:48915}(done)

Noticed IPv4 change for eth0.
Upped interface eth0 (cost=96, channel=-2, IPv4).
Sending hello 34789 (400) to eth0.
Sending self update to eth0.
Sending update to eth0 for any.
Sending self update to eth0.
Sending update to eth0 for any.
sending request to eth0 for any.
sending request to eth0 for any specific.
Sending seqno 48916 from address 0x61b070 (dump)
Netlink message: [multi] {seq:48916}(msg -> "found address on interface lo(1): 127.0.0.1
" 0), [multi] {seq:48916}(msg -> "found address on interface eth0(42): 172.17.0.2
" 0),
Netlink message: [multi] {seq:48916}(msg -> "found address on interface lo(1): ::1
" 0), [multi] {seq:48916}(msg -> "found address on interface eth0(42): 2001:db8:1::242:ac11:2
" 0), [multi] {seq:48916}(msg -> "found address on interface eth0(42): fe80::42:acff:fe11:2
" 1),
Netlink message: [multi] {seq:48916}(done)


Checking kernel routes.
Sending seqno 48917 from address 0x61b070 (dump)
Netlink message: [multi] {seq:48917}(msg -> "found address on interface lo(1): 127.0.0.1
" 1), [multi] {seq:48917}(msg -> "found address on interface eth0(42): 172.17.0.2
" 1),
Netlink message: [multi] {seq:48917}(msg -> "found address on interface lo(1): ::1
" 1), [multi] {seq:48917}(msg -> "found address on interface eth0(42): 2001:db8:1::242:ac11:2

[...]
```

Now run a second one, and check that you have the first container in the routing table, and you can ping it:

```
$ docker run -d --privileged zoobab/babeld-in-docker
f203f3cecca71122704160e61e1df9b537f745bee0b091b90be5a95598c3afb6
$ docker run -d --privileged zoobab/babeld-in-docker
342d63b63798de48b9f07049f78bd12f003a9cee58a411e40338a91da2b8576f
$ docker ps
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS              PORTS                    NAMES
342d63b63798        zoobab/babeld-in-docker   "babeld -d 9 eth0"       37 seconds ago      Up 35 seconds                                compassionate_ellis
f203f3cecca7        zoobab/babeld-in-docker   "babeld -d 9 eth0"       38 seconds ago      Up 37 seconds                                xenodochial_raman
```

Check the routing table:

```
$ docker exec -it 342d63b63798 /sbin/route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.17.0.6      172.17.0.6      255.255.255.255 UGH   0      0        0 eth0
$ docker exec -it f203f3cecca7 /sbin/route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.17.0.7      172.17.0.7      255.255.255.255 UGH   0      0        0 eth0
```

Try pinging the other container:

```
$ docker exec -it 342d63b63798 ping -c1 172.17.0.6
PING 172.17.0.6 (172.17.0.6): 56 data bytes
64 bytes from 172.17.0.6: seq=0 ttl=64 time=0.089 ms

--- 172.17.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.089/0.089/0.089 ms
```

Run in Kubernetes
=================

I run a minimal kubernetes cluster with K3D from the K3S project of Rancher (https://github.com/rancher/k3d), which just need Docker running:

```
# /usr/local/bin/k3d c --name="babeldcluster"
2019/07/09 09:40:18 Created cluster network with ID 4bb9042dc2805c65e150c40716a1b9eb54c5e196ef6e749af222cb865f541452
2019/07/09 09:40:18 Creating cluster [babeldcluster]
2019/07/09 09:40:18 Creating server using docker.io/rancher/k3s:v0.5.0...
2019/07/09 09:40:18 SUCCESS: created cluster [babeldcluster]
2019/07/09 09:40:18 You can now use the cluster with:

export KUBECONFIG="$(/usr/local/bin/k3d get-kubeconfig --name='babeldcluster')"
kubectl cluster-info
```

You can add the export line in your ```.bashrc```, reload bash, and you get be able to get the cluster-status:

```
$ kubectl cluster-info
Kubernetes master is running at https://localhost:6443
CoreDNS is running at https://localhost:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Now try to run a 2 pods with babeld:

```
$ kubectl apply -f ./babeld1.yaml
pod/babeld1 created
$ kubectl apply -f ./babeld2.yaml
pod/babeld2 created
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
babeld1   1/1       Running   0          12s
babeld2   1/1       Running   0          8s
$ kubectl get pods -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP          NODE                       NOMINATED NODE   READINESS GATES
babeld1   1/1       Running   0          2m2s      10.42.0.6   k3d-babeldcluster-server   <none>           <none>
babeld2   1/1       Running   0          118s      10.42.0.7   k3d-babeldcluster-server   <none>           <none>
```

Check the routing tables:

```
$ kubectl exec -it babeld1 -- route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.42.0.1       0.0.0.0         UG    0      0        0 eth0
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.42.0.0       10.42.0.1       255.255.0.0     UG    0      0        0 eth0
10.42.0.7       10.42.0.7       255.255.255.255 UGH   0      0        0 eth0
$ kubectl exec -it babeld2 -- route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.42.0.1       0.0.0.0         UG    0      0        0 eth0
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.42.0.0       10.42.0.1       255.255.0.0     UG    0      0        0 eth0
10.42.0.6       10.42.0.6       255.255.255.255 UGH   0      0        0 eth0
```

Now try to ping from babeld1 to babeld2:

```
$ kubectl  exec -it babeld1 -- ping -c1 10.42.0.7
PING 10.42.0.7 (10.42.0.7): 56 data bytes
64 bytes from 10.42.0.7: seq=0 ttl=64 time=0.122 ms

--- 10.42.0.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.122/0.122/0.122 ms
```

And the reverse from babeld2 to babeld1:

```
$ kubectl  exec -it babeld2 -- ping -c1 10.42.0.6
PING 10.42.0.6 (10.42.0.6): 56 data bytes
64 bytes from 10.42.0.6: seq=0 ttl=64 time=0.093 ms

--- 10.42.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.093/0.093/0.093 ms
```

Now with kubectl run with 10 replicas:

```
$ kubectl run babeld10 --image=zoobab/babeld-in-docker --replicas=10 --overrides='{"spec": {"template": {"spec": {"containers": [{"name": "babeld10", "image": "zoobab/babeld-in-docker", "securityContext": {"privileged": true} }]}}}}' --dry-run -o yaml > babeld10.yaml
$ kubectl apply -f ./babeld10.yaml
$ kubectl  get pods
NAME                       READY     STATUS    RESTARTS   AGE
babeld1                    1/1       Running   0          7m23s
babeld10-f46679bf7-5q9kp   1/1       Running   0          83s
babeld10-f46679bf7-67zsl   1/1       Running   0          83s
babeld10-f46679bf7-clp46   1/1       Running   0          83s
babeld10-f46679bf7-gtmrf   1/1       Running   0          83s
babeld10-f46679bf7-gwx9p   1/1       Running   0          83s
babeld10-f46679bf7-hhc7h   1/1       Running   0          83s
babeld10-f46679bf7-jsdc4   1/1       Running   0          83s
babeld10-f46679bf7-ll72x   1/1       Running   0          83s
babeld10-f46679bf7-mdjjq   1/1       Running   0          83s
babeld10-f46679bf7-w6r2p   1/1       Running   0          83s
babeld2                    1/1       Running   0          7m19s
$ kubectl  get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP           NODE                       NOMINATED NODE   READINESS GATES
babeld1                    1/1       Running   0          7m28s     10.42.0.6    k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-5q9kp   1/1       Running   0          88s       10.42.0.9    k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-67zsl   1/1       Running   0          88s       10.42.0.8    k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-clp46   1/1       Running   0          88s       10.42.0.10   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-gtmrf   1/1       Running   0          88s       10.42.0.13   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-gwx9p   1/1       Running   0          88s       10.42.0.14   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-hhc7h   1/1       Running   0          88s       10.42.0.11   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-jsdc4   1/1       Running   0          88s       10.42.0.16   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-ll72x   1/1       Running   0          88s       10.42.0.17   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-mdjjq   1/1       Running   0          88s       10.42.0.15   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-w6r2p   1/1       Running   0          88s       10.42.0.12   k3d-babeldcluster-server   <none>           <none>
babeld2                    1/1       Running   0          7m24s     10.42.0.7    k3d-babeldcluster-server   <none>           <none>
```

Scale up to 20 nodes:

```
$ kubectl scale --replicas=20 deployments/babeld10
deployment.extensions/babeld10 scaled
$ kubectl  get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP           NODE                       NOMINATED NODE   READINESS GATES
babeld10-f46679bf7-5q9kp   1/1       Running   0          6m5s      10.42.0.9    k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-67zsl   1/1       Running   0          6m5s      10.42.0.8    k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-6hj7l   1/1       Running   0          2m19s     10.42.0.28   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-87b8c   1/1       Running   0          2m19s     10.42.0.31   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-9x4kq   1/1       Running   0          2m19s     10.42.0.35   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-clp46   1/1       Running   0          6m5s      10.42.0.10   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-ddrdr   1/1       Running   0          2m19s     10.42.0.38   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-gtmrf   1/1       Running   0          6m5s      10.42.0.13   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-gwx9p   1/1       Running   0          6m5s      10.42.0.14   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-hcxf8   1/1       Running   0          2m19s     10.42.0.26   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-hhc7h   1/1       Running   0          6m5s      10.42.0.11   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-j5fwj   1/1       Running   0          2m19s     10.42.0.33   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-jsdc4   1/1       Running   0          6m5s      10.42.0.16   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-kpl55   1/1       Running   0          2m19s     10.42.0.27   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-ll72x   1/1       Running   0          6m5s      10.42.0.17   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-mdjjq   1/1       Running   0          6m5s      10.42.0.15   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-rcvk8   1/1       Running   0          2m19s     10.42.0.36   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-tb8zc   1/1       Running   0          2m19s     10.42.0.44   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-vlcv9   1/1       Running   0          2m19s     10.42.0.37   k3d-babeldcluster-server   <none>           <none>
babeld10-f46679bf7-w6r2p   1/1       Running   0          6m5s      10.42.0.12   k3d-babeldcluster-server   <none>           <none>
```
