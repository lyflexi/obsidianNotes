场景：K8s v1.17 三节点 master 群集，其中一个节点因意外系统故障。

可以通过以下命令模拟故障

```shell
kubeadm reset -f
```

在其他 master 节点上删除故障节点

```shell
kubectl drain k8sm01 --delete-local-data --force --ignore-daemonsets #报错直接执行下一条命令
kubectl delete node k8sm01 #k8sm01为故障节点的node名称
```

重建 k8sm01 节点，相同的系统配置，加入群集，报错如下：

原因是该节点虽然已经删除或者重置，但 ETCD 数据库中的节点信息任然存储，重新加入群集时检测到 ETCD 节点健康状态有问题，无法继续。

```shell
[check-etcd] Checking that the etcd cluster is healthy
error execution phase check-etcd: etcd cluster is not healthy: failed to dial endpoint https://10.19.188.101:2379 with maintenance client: context deadline exceeded
To see the stack trace of this error execute with --v=5 or higher
```

解决方法是获取故障节点的 ETCD member id

```shell
ETCD=`docker ps|grep etcd|grep -v POD|awk '{print $1}'`
docker exec \
  -it ${ETCD} \
  etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/peer.crt \
  --key /etc/kubernetes/pki/etcd/peer.key \
  member list
```

输出如下：

```shell
3d3f3c689bf33274, started, k8sm03, https://10.19.188.103:2380, https://10.19.188.103:2379, false
6b8df7d4fba17b13, started, k8sm02, https://10.19.188.102:2380, https://10.19.188.102:2379, false
e66f819c0e1f96eb, started, k8sm01, https://10.19.188.101:2380, https://10.19.188.101:2379, false
```

本例中的故障 member ID 为 e66f819c0e1f96eb，由于故障节点已经被删除，因此相当于该 ID 对应的 ETCD 实例已经丢失，无法通信。

运行以下命令将故障的 member 从 etcd 集群中删除：

```shell
ETCD=`docker ps|grep etcd|grep -v POD|awk '{print $1}'`
docker exec \
  -it ${ETCD} \
  etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/peer.crt \
  --key /etc/kubernetes/pki/etcd/peer.key \
  member remove e66f819c0e1f96eb
```

输出如下：

```shell
{"level":"warn","ts":"2020-05-01T10:13:51.466Z","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-63e34676-017c-4963-91e0-7b46a100d438/127.0.0.1:2379","attempt":0,"error":"rpc error: code = NotFound desc = etcdserver: member not found"}
Error: etcdserver: member not found
```

虽然报错，再次运行上述 member list 查看已经移除成功。

在新的 k8sm01 节点上执行’kubeadm reset’，再次加入群集，成功！！！

```shell
# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
k8sm01   Ready    master   3m22s   v1.17.0
k8sm02   Ready    master   117d    v1.17.0
k8sm03   Ready    master   117d    v1.17.0
k8sn01   Ready    <none>   117d    v1.17.0
k8sn02   Ready    <none>   117d    v1.17.0
k8sn03   Ready    <none>   152m    v1.17.0
```

命令帮助，查看帮助：

```shell
ETCD=`docker ps|grep etcd|grep -v POD|awk '{print $1}'`
docker exec \
  -it ${ETCD} \
  etcdctl \
  --endpoints https://127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/peer.crt \
  --key /etc/kubernetes/pki/etcd/peer.key \
  help
```

输出如下，注意不同版本的命令和参数有所差异

```shell
NAME:
        etcdctl - A simple command line client for etcd3.

USAGE:
        etcdctl [flags]

VERSION:
        3.4.3

API VERSION:
        3.4


COMMANDS:
        alarm disarm            Disarms all alarms
        alarm list              Lists all alarms
        auth disable            Disables authentication
        auth enable             Enables authentication
        check datascale         Check the memory usage of holding data for different workloads on a given server endpoint.
        check perf              Check the performance of the etcd cluster
        compaction              Compacts the event history in etcd
        defrag                  Defragments the storage of the etcd members with given endpoints
        del                     Removes the specified key or range of keys [key, range_end)
        elect                   Observes and participates in leader election
        endpoint hashkv         Prints the KV history hash for each endpoint in --endpoints
        endpoint health         Checks the healthiness of endpoints specified in `--endpoints` flag
        endpoint status         Prints out the status of endpoints specified in `--endpoints` flag
        get                     Gets the key or a range of keys
        help                    Help about any command
        lease grant             Creates leases
        lease keep-alive        Keeps leases alive (renew)
        lease list              List all active leases
        lease revoke            Revokes leases
        lease timetolive        Get lease information
        lock                    Acquires a named lock
        make-mirror             Makes a mirror at the destination etcd cluster
        member add              Adds a member into the cluster
        member list             Lists all members in the cluster
        member promote          Promotes a non-voting member in the cluster
        member remove           Removes a member from the cluster
        member update           Updates a member in the cluster
        migrate                 Migrates keys in a v2 store to a mvcc store
        move-leader             Transfers leadership to another etcd cluster member.
        put                     Puts the given key into the store
        role add                Adds a new role
        role delete             Deletes a role
        role get                Gets detailed information of a role
        role grant-permission   Grants a key to a role
        role list               Lists all roles
        role revoke-permission  Revokes a key from a role
        snapshot restore        Restores an etcd member snapshot to an etcd directory
        snapshot save           Stores an etcd node backend snapshot to a given file
        snapshot status         Gets backend snapshot status of a given file
        txn                     Txn processes all the requests in one transaction
        user add                Adds a new user
        user delete             Deletes a user
        user get                Gets detailed information of a user
        user grant-role         Grants a role to a user
        user list               Lists all users
        user passwd             Changes password of user
        user revoke-role        Revokes a role from a user
        version                 Prints the version of etcdctl
        watch                   Watches events stream on keys or prefixes

OPTIONS:
      --cacert=""                               verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""                                 identify secure client using this TLS certificate file
      --command-timeout=5s                      timeout for short running command (excluding dial timeout)
      --debug[=false]                           enable client-side debug logging
      --dial-timeout=2s                         dial timeout for client connections
  -d, --discovery-srv=""                        domain name to query for SRV records describing cluster endpoints
      --discovery-srv-name=""                   service name to query when using DNS discovery
      --endpoints=[127.0.0.1:2379]              gRPC endpoints
  -h, --help[=false]                            help for etcdctl
      --hex[=false]                             print byte strings as hex encoded strings
      --insecure-discovery[=true]               accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]        skip server certificate verification
      --insecure-transport[=true]               disable transport security for client connections
      --keepalive-time=2s                       keepalive time for client connections
      --keepalive-timeout=6s                    keepalive timeout for client connections
      --key=""                                  identify secure client using this TLS key file
      --password=""                             password for authentication (if this option is used, --user option shouldn't include password)
      --user=""                                 username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"                      set the output format (fields, json, protobuf, simple, table)
```