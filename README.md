![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)


# kubespray
이 레포지토리는 kubespray를 이용하여 kubernetes를 쉽게 설치하기 위한 레포지토리이다. 여기서 사용한 노드는 Cpu: 2core, Ram: 4GB의 5개의 노드로 구성되어 있으며 3~4개의 노드에서도 잘 동작한다. 이 설치는 다음 레포지토리([kubernetes-tutorial](https://github.com/stevejhkang/kubernetes-tutorial))와 밀집하게 연관이 되어 있다.

### Requirement 설치

```
#### git과  파이썬3.6 설치
```bash
sudo yum update
sudo yum install -y git

sudo yum install -y https://centos7.iuscommunity.org/ius-release.rpm
sudo yum install -y python3 python3-pip python-devel
```

### kubespray로 kubernetes 설치
#### kubespary를 이용하기 위해 설치하는 호스트에서 다른 호스트로  ssh접속 가능하게 하기
```bash
# 파이썬 디펜던시 설치
cd kubespray
sudo pip3 install -r requirements.txt 

# 1. 쿠버네티스를 설치할 서버의 IP들을 선언해주고 노드에 접속할 수 있는 private키를 환경변수로 지정(설치 후 삭제 필요)
declare -a IPS=(192.168.0.125 192.168.0.126 192.168.0.127 192.168.0.128 192.168.0.129)
PEM_KEY=/private키를/경로로/지정해/주세요.pem
echo $PEM_KEY

# 2. kubespary 설치 과정 중 다른 호스트에 접속하기 위한 키 생성
ssh-keygen -t rsa -b 4096
export SSH_PUBKEY=$HOME/.ssh/id_rsa.pub

# 3. 각 호스트에 ssh접속을 위한 public키를 복사하고, kubernetes 설치를 위해 swap기능을 끈다.
for i in ${IPS[@]}
do
  ssh-keyscan -H $i >> $HOME/.ssh/known_hosts
  scp -i $PEM_KEY $SSH_PUBKEY  $i:$HOME
  ssh -i $PEM_KEY $i  "cat $HOME/id_rsa.pub >> $HOME/.ssh/authorized_keys"
  ssh $i "sudo swapoff -a"
done
```

#### 파일복사 및 설정파일 자동 설정 명령
```bash
cp -r inventory/sample inventory/mycluster
CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

#### kubespray/inventory/mycluster/hosts.ini에서 설정 조정
마스터 두개로 하고 싶으면 [kube-master]안에 노드 추가
마스터와 노드를 동시에 할당하고 싶으면 [kube-node]와 [kube-master]에 둘다 추가


```ini
[all]
node1    ansible_host=192.168.0.125 ip=192.168.0.125
node2    ansible_host=192.168.0.126 ip=192.168.0.126
node3    ansible_host=192.168.0.127 ip=192.168.0.127
node4    ansible_host=192.168.0.128 ip=192.168.0.128
node5    ansible_host=192.168.0.129 ip=192.168.0.129

[kube-master]
node1
node2

[etcd]
node1
node2
node3

[kube-node]
node1
node2
node3
node4
node5

[k8s-cluster:children]
kube-master
kube-node

[calico-rr]
```

#### inventory/mycluster/k8s-cluster/k8s-cluster.yml에서 network plugin 수정
kube_network_plugin: callico -> weave로 변경

```yaml
# Directory where credentials will be stored
credentials_dir: "{{ inventory_dir }}/credentials"

# Users to create for basic auth in Kubernetes API via HTTP
# Optionally add groups for user
kube_api_pwd: "{{ lookup('password', credentials_dir + '/kube_user.creds length=15 chars=ascii_letters,digits') }}"
kube_users:
  kube:
    pass: "{{kube_api_pwd}}"
    role: admin
    groups:
      - system:masters

## It is possible to activate / deactivate selected authentication methods (basic auth, static token auth)
#kube_oidc_auth: false
#kube_basic_auth: false
#kube_token_auth: false


## Variables for OpenID Connect Configuration https://kubernetes.io/docs/admin/authentication/
## To use OpenID you have to deploy additional an OpenID Provider (e.g Dex, Keycloak, ...)

# kube_oidc_url: https:// ...
# kube_oidc_client_id: kubernetes
## Optional settings for OIDC
# kube_oidc_ca_file: "{{ kube_cert_dir }}/ca.pem"
# kube_oidc_username_claim: sub
# kube_oidc_username_prefix: oidc:
# kube_oidc_groups_claim: groups
# kube_oidc_groups_prefix: oidc:


# Choose network plugin (cilium, calico, contiv, weave or flannel)
# Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
kube_network_plugin: weave

# Setting multi_networking to true will install Multus: https://github.com/intel/multus-cni
kube_network_plugin_multus: false

# Kubernetes internal network for services, unused block of space.
kube_service_addresses: 10.233.0.0/18

# internal network. When used, it will assign IP
# addresses from this range to individual pods.
```

#### inventory/mycluster/k8s-cluster/addons.yml에서 helm과 metric-server 설정

helm_enabled: false -> true

metrics_server_enabled: false -> true 시켜주고 주석해제

```yaml
# Kubernetes dashboard
# RBAC required. see docs/getting-started.md for access details.
dashboard_enabled: true

# Helm deployment
helm_enabled: true

# Registry deployment
registry_enabled: false
# registry_namespace: kube-system
# registry_storage_class: ""
# registry_disk_size: "10Gi"

# Metrics Server deployment
metrics_server_enabled: true
metrics_server_kubelet_insecure_tls: true
metrics_server_metric_resolution: 60s
metrics_server_kubelet_preferred_address_types: "InternalIP"

# Local volume provisioner deployment
local_volume_provisioner_enabled: false
# local_volume_provisioner_namespace: kube-system
# local_volume_provisioner_storage_classes:
#   local-storage:
#     host_dir: /mnt/disks
#     mount_dir: /mnt/disks
#   fast-disks:
#     host_dir: /mnt/fast-disks
#     mount_dir: /mnt/fast-disks
#     block_cleaner_command:
#       - "/scripts/shred.sh"
#       - "2"
#     volume_mode: Filesystem
#     fs_type: ext4

# CephFS provisioner deployment
cephfs_provisioner_enabled: false
# cephfs_provisioner_namespace: "cephfs-provisioner"
# cephfs_provisioner_cluster: ceph
# cephfs_provisioner_monitors: "172.24.0.1:6789,172.24.0.2:6789,172.24.0.3:6789"
# cephfs_provisioner_admin_id: admin
# cephfs_provisioner_secret: secret
# cephfs_provisioner_storage_class: cephfs
# cephfs_provisioner_reclaim_policy: Delete
# cephfs_provisioner_claim_root: /volumes
# cephfs_provisioner_deterministic_names: true
```



#### ansible-playbook을 이용해 kubespray 설치 명령

--become --become-user=root 옵션을 줌으로써 설치하는 동안 root권한 임시 사용

```bash
ansible-playbook -i inventory/mycluster/hosts.ini --become --become-user=root cluster.yml
```

#### kube admin에 관한 환경변수 설정

```bash
kubectl get nodes 
```

했을시 다음과 같은 에러가 나면 Kubernetes error: “The connection to the server localhost:8080 was refused – did you specify the right host or port ?”

kube admin에 관한 환경변수를 설정해주어야 한다.

```bash
sudo cp /etc/kubernetes/admin.conf $HOME
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
echo "export KUBECONFIG=$HOME/admin.conf" > $HOME/.bashrc
```
#### 자동완성과 kubectl 대신 약어(k) 사용하는 방법
https://kubernetes.io/docs/tasks/tools/install-kubectl/
<https://kubernetes.io/ko/docs/reference/kubectl/cheatsheet/>

#### Defendencies
* os: centos 7.5
* docker version: 18.09.2
* go version: 1.10.6
* kubernetes version: 1.13.3
* kubespray: 2.8.3

