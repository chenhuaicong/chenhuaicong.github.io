# openshift节点扩容指导手册
## 初始化配置
### 1.配置hosts，并同步到所有节点，并重启dnsmasq
```
ansible all -m copy -a 'src=/etc/hosts dest=/etc/hosts'
#重启节点dnsmasq
ansible nodes -m shell -a "systemctl restart dnsmasq "
```


### 2.配置密码，ssh-key
```
echo "xxxxxxx" | passwd --stdin root 


cat > /root/.ssh/authorized_keys << EOF
xxxxxxxxxxxxxxxxxxxxx
EOF

chmod 0600 /root/.ssh/authorized_keys
```

### 3.关闭swap分区
（1）临时关闭swap分区, 重启失效; 
```
swapoff -a
```
（2）永久关闭swap分区
```
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 4.开启selinux 并设置为permissive模式
```
sed -i 's/SELINUX=disabled/SELINUX=permissive/g' /etc/sysconfig/selinux
```


### 5.修改grup ，启用ipv6 模块

```
sed -i 's/ipv6.disable=1/ipv6.disable=0/g' /etc/default/grub

# 生成新的grub菜单
grub2-mkconfig -o /boot/grub2/grub.cfg
```

```
cat /etc/default/grub
...
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root ipv6.disable=0 rd.lvm.lv=centos/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
...

grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 6.重启系统

### 7.配置yum 源
```
cd /etc/yum.repos.d/
scp centos.repo ceph.repo epel.repo CentOS-OpenShift-Origin311.repo szpbs-okd-prd-node$i:/etc/yum.repos.d/
```
### 8.安装必要的软件包
```
yum install -y wget rsync vim net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
```

---
## docker安装配置
### 安装docker-1.13版本
```
yum install -y docker-1.13.1 >/dev/null 2>&1
```


### 配置docker 使用的存储
删除以前安装的docker和配置
```
lvremove -f /dev/docker-vg/docker-pool
vgremove docker-vg
pvremove /dev/xvdf1
wipefs -af /dev/xvdf
```

查看存储配置文件： /etc/sysconfig/docker-storage-setup
```
STORAGE_DRIVER=overlay2
DEVS=/dev/vdb
VG=docker-vg
CONTAINER_ROOT_LV_NAME=dockerlv
CONTAINER_ROOT_LV_SIZE=100%FREE
# 下面这个mount的路径就比较特殊了，其实这个相关的配置文件就是使用sdb创建一个pv、vg、lv，然后格式化成xfs，最后mount一下，docker只是去使用这个路径下空间，如果docker存储的路径需要改变，这个mount的路径也需要进行修改。
CONTAINER_ROOT_LV_MOUNT_PATH=/var/lib/docker
```

执行
```
docker-storage-setup   
```
### 配置国内镜像加速器
对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）
```
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com"
  ]
}
```
启动docker
```
systemctl restart docker;systemctl enable docker
```
---
## 配置openshift
### 配置openshift本地volume卷
```
parted -s /dev/sdc mklabel gpt
parted -s /dev/sdc mkpart primary xfs 1 215GB
pvcreate /dev/sdc1
vgcreate openshift-local-volumes /dev/sdc1
lvcreate -n openshift -l 100%FREE openshift-local-volumes
mkfs.xfs /dev/mapper/openshift--local--volumes-openshift
mkdir /var/lib/origin/openshift.local.volumes -p
mount /dev/mapper/openshift--local--volumes-openshift /var/lib/origin/openshift.local.volumes
echo "/dev/openshift-local-volumes/openshift /var/lib/origin/openshift.local.volumes xfs defaults 0 0" >>/etc/fstab
```

### 配置inventory文件 /etc/ansible/hosts
新增new_nodes 分组
```
[OSEv3:children]
masters
nodes
etcd
#新增new_nodes 分组
new_nodes
```
在new_nodes分组下面配置要扩容的节点
```
[new_nodes]
szpbs-okd-prd-node[11:20] openshift_node_group_name='node-config-compute'
```

导入本地镜像

```
docker load -i <image_version>.tar
```

执行脚本扩容节点
```
ansible-playbook /opt/openshift-ansible/playbooks/openshift-node/scaleup.yml
```

为节点添加添加log-pilot-filebeat=true 标签，使得log-pilot 日志收集组件部署到新到节点上去
```
oc  label nodes szpbs-okd-prd-node7 log-pilot-filebeat=true
```