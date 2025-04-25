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