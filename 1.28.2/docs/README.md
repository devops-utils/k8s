```shell
sudo yum install docker-ce
sudo yum install -y kubectl kubeadm kubelet

sudo systemctl enable kubelet
sudo systemctl status kubelet
sudo systemctl start kubelet
sudo systemctl restart kubelet

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