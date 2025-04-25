```shell
sudo yum install docker-ce
sudo yum install -y kubectl kubeadm kubelet

sudo systemctl enable kubelet
sudo systemctl status kubelet
sudo systemctl start kubelet
sudo systemctl restart kubelet

sudo kubeadm reset -f
sudo docker system prune

docker rmi $(docker images -f "dangling=true" -q)
docker rmi $(docker images -q)

journalctl -xeu kubelet
ls /var/lib/kubelet/

containerd config default > /etc/containerd/config.toml
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
systemctl restart containerd.service

kubeadm config images pull --image-repository  registry.aliyuncs.com/google_containers

kubeadm init --kubernetes-version=1.28.2 \
--apiserver-advertise-address=10.50.10.20  \
--image-repository  registry.aliyuncs.com/google_containers \
--pod-network-cidr=172.16.0.0/16

kubeadm token create --print-join-command
kubeadm join 10.50.10.20:6443 --token wdqk5e.2yt122t2znscjmw4 --discovery-token-ca-cert-hash sha256:92adc2d9849b559af297f630785bb1681ca652beb0285c19a1175a23af9ea438


kubeadm join 10.50.10.20:6443 --token rasuu7.xw06c5dkyjievo6j \
        --discovery-token-ca-cert-hash sha256:92adc2d9849b559af297f630785bb1681ca652beb0285c19a1175a23af9ea438

kubectl get nodes

kubectl get pods -A
kubectl describe pods -n kube-system coredns-66f779496c-rwskb

kubectl delete pod coredns-66f779496c-rwskb -n kube-system

kubectl apply -f calico.yaml

kubectl get nodes h1.corp.sz.gene.com --show-labels
kubectl label nodes h1.corp.sz.gene.com node-role.kubernetes.io/master=
node/h1.corp.sz.gene.com labeled

kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml
kubectl get pod -n kuboard -o wide
kubectl describe pods -n kuboard kuboard-etcd-jrn9m
The node had condition: [DiskPressure].
kubectl delete pod -n kuboard kuboard-etcd-26csd

systemctl cat kubelet
vim /var/lib/kubelet/config.yaml
vim /etc/sysconfig/kubelet

http://your-node-ip-address:30080
admin
Kuboard123
```

```shell
curl -k 'http://10.50.10.27:30080/kuboard-api/cluster/default/kind/KubernetesCluster/default/resource/installAgentToKubernetes?token=YG1wYUw4gwKjOd48m0CKdWLHQ3xpFB6t' > kuboard-agent.yaml
kubectl apply -f ./kuboard-agent.yaml
```

```shell
清理方法
1 自动清理命令
docker system prune可对空间进行自动清理。
该命令所清理的对象如下：
已停止的容器
未被任何容器使用的卷
未被任何容器所关联的网络
所有悬空的镜像
对于上面提到的一些镜像或容器的状态，需要我们心里有点数：
已使用的镜像：指所有已被容器（包括stop的）关联的镜像，也就是docker ps -a所看到的所有容器对应的image。
未引用镜像：没有被分配或使用在容器中的镜像
悬空镜像（dangling image）：未配置任何Tag（也就是无法被引用）的镜像。通常是由于镜像编译过程中未指定-t参数配置Tag导致的。
docker system prune后可以加额外的参数，如：
docker system prune -a ： 一并清除所有未被使用的镜像和悬空镜像。
docker system prune -f ： 用以强制删除，不提示信息。
另外除了system级别的，还有针对容器或是镜像级别的删除命令：
docker image prune：删除悬空的镜像。

docker container prune：删除无用的容器。
      --默认情况下docker container prune命令会清理掉所有处于stopped状态的容器
      --如果不想那么残忍统统都删掉，也可以使用--filter标志来筛选出不希望被清理掉的容器。例子：清除掉所有停掉的容器，但24内创建的除外：
      --$ docker container prune --filter "until=24h"  

docker volume prune：删除无用的卷。
docker network prune：删除无用的网络

手动清除
对于悬空镜像和未使用镜像可以使用手动进行个别删除：
1、删除所有悬空镜像，不删除未使用镜像：docker rmi $(docker images -f "dangling=true" -q)
2、删除所有未使用镜像和悬空镜像
docker rmi $(docker images -q)
3、清理卷
如果卷占用空间过高，可以清除一些不使用的卷，包括一些未被任何容器调用的卷（-v 详细信息中若显示 LINKS = 0，则是未被调用）：
删除所有未被容器引用的卷：
docker volume rm $(docker volume ls -qf dangling=true)
4、容器清理
如果发现是容器占用过高的空间，可以手动删除一些：
删除所有已退出的容器：
docker rm -v $(docker ps -aq -f status=exited)
删除所有状态为dead的容器
docker rm -v $(docker ps -aq -f status=dead)
```

```shell
kubelet 压力驱逐 - The node had condition：[DiskPressure]
文章目录
浅言碎语
清理 docker
kubelet 节点压力驱逐
驱逐信号
驱逐条件
软驱逐条件
硬驱逐条件
驱逐监测间隔
节点条件
最小驱逐回收
修改 kubelet 配置文件
修改 kubelet 配置文件的方式
查看配置文件路径
备份配置文件
修改 kubelet.service 的方式
查看配置文件路径
备份配置文件
重启 kubelet
浅言碎语
在查看 pod 运行状态时，发现有的 pod 的状态是 Evicted，通过 describe 去查看发现了 The node had condition: [DiskPressure]. 的报错

原因是 kubelet 检测到本地磁盘使用率超过了 85% ，这是 kubelet 的默认配置

grep '85%' /var/log/messages

可以看到类似如下的内容

May 7 10:45:19 DC3-03-010 kubelet[637219]: I0507 10:45:19.660844 637219 image_gc_manager.go:300] [imageGCManager]: Disk usage on image filesystem is at 85% which is over the high threshold (85%). Trying to free 4934070272 bytes down to the low threshold (80%).

方法只有两种，要么清理磁盘或者磁盘扩容，要么就是修改 kubelet 的配置

清理 docker
docker system prune 命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及 dangling 的镜像（docker images 命令输出的 REPOSITORY 和 TAG 显示为 <none> 的镜像）
docker system prune -a 命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉
这两个命令会将当前关闭的容器，以及当前没有用到的 Docker 镜像都删掉了
所以使用之前一定要确认清楚了
docker system prune
docker system prune -a

kubelet 节点压力驱逐
kubelet 监控集群节点的 CPU、内存、磁盘空间和文件系统的 inode 等资源
当这些资源中的一个或者多个达到特定的消耗水平， kubelet 可以主动地使节点上一个或者多个 Pod 失效，以回收资源防止饥饿
在节点压力驱逐期间，kubelet 将所选 Pod 的 PodPhase 设置为 Failed。这将终止 Pod
kubelet 在终止最终用户 Pod 之前会尝试回收节点级资源
例如，它会在磁盘资源不足时删除未使用的容器镜像
驱逐信号
memory.available
当前节点可用内存
计算方式为 cgroup memory 子系统中 memory.usage_in_bytes 中的值减去 memory.stat 中 total_inactive_file 的值
默认值为 memory.available<100Mi
nodefs.available
nodefs 包含 kubelet 配置中 --root-dir 指定的文件分区和 /var/lib/kubelet/ 所在的分区磁盘使用率
默认值为 nodefs.available<10%
nodefs.inodesFree
nodefs.available 分区的 inode 使用率
imagefs.available
镜像所在分区磁盘使用率
默认值为 imagefs.available<15%
imagefs.inodesFree
镜像所在分区磁盘 inode 使用率
默认值为 nodefs.inodesFree<5% （Linux 节点）
pid.available
/proc/sys/kernel/pid_max 中的值为系统最大可用 pid 数
kubelet 可以通过参数 --eviction-hard 来配置以上几个参数的阈值，当达到阈值时会驱逐节点上的容器
驱逐条件
驱逐条件的形式为 [eviction-signal][operator][quantity]
eviction-signal 是要使用的驱逐信号
operator 是你想要的关系运算符， 比如 <（小于）
quantity 是驱逐条件数量，例如 1Gi
quantity 的值必须与 Kubernetes 使用的数量表示相匹配
可以使用文字值或百分比（%）
例如，如果一个节点的总内存为 10Gi 并且你希望在可用内存低于 1Gi 时触发驱逐
可以将驱逐条件定义为 memory.available<10%
或者 memory.available<1G
不能同时使用二者
软驱逐条件
将驱逐条件与管理员所必须指定的宽限期配对
在超过宽限期之前，kubelet 不会驱逐 Pod
如果没有指定的宽限期，kubelet 会在启动时返回错误
可以既指定软驱逐条件宽限期，又指定 Pod 终止宽限期的上限，给 kubelet 在驱逐期间使用
如果指定了宽限期的上限并且 Pod 满足软驱逐阈条件，则 kubelet 将使用两个宽限期中的较小者
如果没有指定宽限期上限，kubelet 会立即杀死被驱逐的 Pod，不允许其体面终止
软驱逐条件标志
eviction-soft：一组驱逐条件，如 memory.available<1.5Gi， 如果驱逐条件持续时长超过指定的宽限期，可以触发 Pod 驱逐。
eviction-soft-grace-period：一组驱逐宽限期， 如 memory.available=1m30s，定义软驱逐条件在触发 Pod 驱逐之前必须保持多长时间。
eviction-max-pod-grace-period：在满足软驱逐条件而终止 Pod 时使用的最大允许宽限期（以秒为单位）
硬驱逐条件
硬驱逐条件没有宽限期
当达到硬驱逐条件时， kubelet 会立即杀死 pod，而不会正常终止以回收紧缺的资源
硬驱逐条件标志
eviction-hard ：一组硬驱逐条件， 如 memory.available<1Gi
驱逐监测间隔
kubelet 根据其配置的 housekeeping-interval 评估驱逐条件
默认为 10s
节点条件
kubelet 报告节点状况以反映节点处于压力之下，因为满足硬或软驱逐条件，与配置的宽限期无关
MemoryPressure
节点上的可用内存已满足驱逐条件
DiskPressure
节点的根文件系统或映像文件系统上的可用磁盘空间和 inode 已满足驱逐条件
PIDPressure
节点上的可用进程标识符已低于驱逐条件 (Linux)
根据配置的 --node-status-update-frequency 更新节点条件
默认为 10s
最小驱逐回收
可以使用 --eviction-minimum-reclaim 标志为每个资源配置最小回收量
当 kubelet 注意到某个资源耗尽时，它会继续回收该资源，直到回收到所指定的数量为止
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "1Gi"
  imagefs.available: "100Gi"
evictionMinimumReclaim:
  memory.available: "0Mi"
  nodefs.available: "500Mi"
  imagefs.available: "2Gi"

memory.available 信号满足驱逐条件， kubelet 会回收资源
直到信号达到 500Mi 的条件
nodefs.available 信号满足驱逐条件， kubelet 会回收资源
直到信号达到 1Gi 的条件， 然后继续回收至少 500Mi 直到信号达到 1.5Gi
imagefs.available 信号满足驱逐条件， kubelet 会回收资源
直到信号达到 100Gi 的条件， 然后继续回收至少 2Gi 直到信号达到 102Gi
所有资源的默认 eviction-minimum-reclaim 都为 0
其他更多细节，可以查看官方文档：节点压力驱逐

修改 kubelet 配置文件
修改 kubelet 配置文件的方式
查看配置文件路径
systemctl cat kubelet

以自己的路径为准，找到相关的 yaml 配置文件，我的是 kubeadm 的方式部署的，看 KUBELET_CONFIG_ARGS=--config 指定的路径即可

备份配置文件
备份做的好，跑路跑的少
cp /var/lib/kubelet/config.yaml{,.bak}
vim /var/lib/kubelet/config.yaml

# 软驱逐 evictionSoft
# 硬驱逐 evictionHard
evictionHard:
  nodefs.available: "5Gi"
  imagefs.available: "5Gi"
# 最小驱逐回收，可以不配置
evictionMinimumReclaim:
  nodefs.available: "500Mi"

修改 kubelet.service 的方式
查看配置文件路径
systemctl cat kubelet

找到 kubelet.service 文件的路径

备份配置文件
备份做的好，跑路跑的少
以自己的路径为准，一般都会有 EnvironmentFile 参数，如果没有，可以加到 Environment 参数后面

cp /etc/sysconfig/kubelet{,.bak}
vim /etc/sysconfig/kubelet

软驱逐必须要设置驱逐宽限期，如果没有配置，kubelet 会启动失败，报错：grace period must be specified for the soft eviction threshold imagefs.available

--eviction-soft=nodefs.available<5%,imagefs.available<5% \
--eviction-soft-grace-period=nodefs.available=2m,imagefs.available=2m \
--eviction-max-pod-grace-period=30

重启 kubelet
systemctl restart kubelet
```