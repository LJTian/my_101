### Create network ns (创建 network namespace)

- 创建目录 并删除之前的 link
```sh
mkdir -p /var/run/netns
find -L /var/run/netns -type l -delete
```

### Start nginx docker with non network mode(启动 nginx 容器设置network为 none)

```sh
docker run --network=none  -d nginx
```

### Check corresponding pid (查询 nginx 的 pid  )

```sh
docker ps|grep nginx #找到对应的MD5值
docker inspect <containerid>|grep -i pid #根据MD5值 查询 pid

"Pid": 876884,
"PidMode": "",
"PidsLimit": null,
```

### Check network config for the container (查询运行进程的网络配置)

```sh
nsenter -t 876884 -n ip a
```

```log
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    valid_lft forever preferred_lft forever
```

### Link network namespace (链接 network namespace)

```sh
export pid=876884
ln -s /proc/$pid/ns/net /var/run/netns/$pid
ip netns list
```
### Check docker bridge on the host(检查主机上面的容器网桥)

```sh
brctl show
ip a
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:35:40:d3:8b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:35ff:fe40:d38b/64 scope link
       valid_lft forever preferred_lft forever
```

### Create veth pair(创建一对虚拟端口)

```sh
ip link add A type veth peer name B
```

### Config A(配置 A)

```sh
brctl addif docker0 A # 使 A 成为 docker0 的一个端口
ip link set A up # 启动 A
```

### Config B(配置 B)

```sh
SETIP=172.17.0.10
SETMASK=16
GATEWAY=172.17.0.1

ip link set B netns $pid
ip netns exec $pid ip link set dev B name eth0
ip netns exec $pid ip link set eth0 up
ip netns exec $pid ip addr add $SETIP/$SETMASK dev eth0
ip netns exec $pid ip route add default via $GATEWAY
```

### Check connectivity(检查链接)

```sh
curl 172.17.0.10
```

```shell
$ nsenter -t 1645171 -n ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
57: eth0@if58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
link/ether 3a:79:ce:fc:2d:66 brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet 172.17.0.10/16 scope global eth0
valid_lft forever preferred_lft forever
```
