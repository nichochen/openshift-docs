# OpenShift企业版安装：单Master集群

|   项目   |   描述               |
|:-------:|-------------------|
|本文目的|本文描述搭建红帽OpenShift容器平台单Master集群的过程。<br>适合用于在没有互联网连接的环境中搭建测试验证使用的OpenShift集群。|
|安装版本|Red Hat OpenShift Container Platform 3.7|
|作者|陈耿 GitHub ID: nichochen|

本文是一篇安装指引，目的并非用于讲解教学。故一些技术细节将不展开详细介绍，请读者见谅。

# 1 安装材料

## 1.1 安装介质

OpenShift的离线环境安装需要提前准备如下的安装介质。

- OpenShift企业版相关RPM包
- OpenShift企业版相关容器镜像  

> 离线安装介质可以联系红帽工程师获取，或自行准备。自行准备的过程请参考本文末尾的附录章节。

## 1.2 主机

### 1.2.1 主机配置要求

本文示例所使用的主机配置如下：

| 名称    | 域名             | CPU        | 内存  | 磁盘 | IP地址| 说明|
|--------|-----------------|-------------| -----| -----|------|----|
| Master|master.example.com<br>nfs.example.com| 64 bit vCPU x 2 | 4 GB RAM  | 40 GB Disk |192.168.172.201|控制节点|
| Node1|node1.example.com| 64 bit vCPU x 4 | 4 GB RAM  | 40 GB Disk |192.168.172.202|计算节点|
| Node2|node2.example.com<br>registry.example.com<br> yum.example.com| 64 bit vCPU x 4 | 4 GB RAM  | 40 GB Disk |192.168.172.203|计算节点及基础服务|

如资源允许，可扩充各类节点的CPU、内存及磁盘、或增加更多的计算节点，获取更好的性能表现。主机可以使用物理机或虚拟机。推荐配置如下：

| 名称    | 域名             | CPU        | 内存  | 磁盘 | IP地址| 说明|
|--------|-----------------|-------------| -----| -----|------|----|
| Master|master.example.com<br>nfs.example.com| 64 bit vCPU x 4 | 16 GB RAM  | 100 GB Disk |192.168.172.201|控制节点|
| Node1|node1.example.com| 64 bit vCPU x 4 | 16 GB RAM  | 100 GB Disk |192.168.172.202|计算节点|
| Node2|node2.example.com<br>registry.example.com<br> yum.example.com| 64 bit vCPU x 4 | 16 GB RAM  | 100 GB Disk |192.168.172.203|计算节点及基础服务|

## 1.2.2 系统要求

操作系统：<font color="red">Red Hat Enterprise Linux 7.3 Minimal</font>安装。

## 1.2.3 网络配置

所有主机需要有独立的IP地址，可被解析的域名。主机的默认网关不可为空。
	
保证所有的主机的主机名可以被正常解析。为简单起见，本文的环境通过编辑各个节点的`/etc/hosts`文件实现。编辑各个主机的`/etc/hosts`文件，添加如下内容：

	192.168.172.201 master.example.com
	192.168.172.202 node1.example.com
	192.168.172.203 node2.example.com	
	192.168.172.203 registry.example.com
	192.168.172.203 yum.example.com
	192.168.172.201 nfs.example.com	

## 1.2.4 时间同步

请确保所有主机节点的时间已经同步。

# 2 安装流程

OpenShift单Master的安装流程包含以下步骤：

- 一、基础服务搭建。搭建安装所依赖的YUM仓库及容器仓库服务。
- 二、准备主机。准备及配置集群所需的主机节点。
- 三、安装前配置。完整安装所需要的前置配置。
- 四、执行安装。配置并执行OpenShift Ansible Playbook。
- 五、安装后配置。按需配置集群。
- 六、验证安装。验证集群成功安装，功能正常。

# 3 搭建基础服务

获取了OpenShift的安装介质后，需要为RPM软件包建立YUM软件仓库服务器，并搭建容器仓库服务，将容器镜像推送至镜像仓库中备用。

## 3.1 搭建YUM仓库服务

将提前获取的RPM软件包拷贝到主机`yum.example.com`的`/opt`目录下。这里假设包如下：

	[root@yum ~]# ls /opt
	rhel-7-fast-datapath-rpms.zip
	rhel-7-server-extras-rpms.zip
	rhel-7-server-ose-3.7.zip
	rhel-7-server-rpms.zip


解压RPM包至主机`yum.example.com`的`/opt/repos`目录。命令如下：

	[root@yum ~]# mkdir -p /opt/repos
	[root@yum ~]# unzip *.zip -d /opt/repos

创建YUM REPO配置文件`/etc/yum.repos.d/local.repo`。文件内容如下：
	
	[rhel-7-server-rpms]
	name=rhel-7-server-rpms
	baseurl=file:///opt/repos/rhel-7-server-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-server-extras-rpms]
	name=rhel-7-server-extras-rpms
	baseurl=file:///opt/repos/rhel-7-server-extras-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-fast-datapath-rpms]
	name=rhel-7-fast-datapath-rpms
	baseurl=file:///opt/repos/rhel-7-fast-datapath-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-server-ose-3.7-rpms]
	name=rhel-7-server-ose-3.7-rpms
	baseurl=file:///opt/repos/rhel-7-server-ose-3.7-rpms
	enabled=1
	gpgcheck=0
	
安装Httpd服务。命令如下：
	
	[root@yum ~]# subscription-manager clean
	[root@yum ~]# yum clean all
	[root@yum ~]# yum -y install httpd;
	
创建Httpd配置文件`/etc/httpd/conf.d/yum.conf`，发布YUM源目录。文件内容如下：

	Alias /repos "/opt/repos"
	<Directory "/opt/repos">
	  Options +Indexes +FollowSymLinks
	Require all granted
	</Directory>
	<Location /repos>
	SetHandler None
	</Location>
	
启动并设置开机自启动。命令如下：

	[root@yum ~]# systemctl enable httpd;
	[root@yum ~]# systemctl restart httpd;

测试环境，为了简化，直接放行数据包。命令如下：

	[root@yum ~]# firewall-cmd --permanent --zone=public --add-service=http
	[root@yum ~]# firewall-cmd --reload
	
> 注意：不可直接停用iptables服务，Docker及Kubernetes依赖iptables服务。
	
安装及配置完Httpd服务后通过访问`http://node2.example.com/repos`可看到RPM软件包目录。

	[root@yum ~]# curl node2.example.com/repos/
	<html>
	<head><title>Index of /repos/</title></head>
	<body bgcolor="white">
	<h1>Index of /repos/</h1><hr><pre><a href="../">../</a>
	<a href="repodata/">repodata/</a>                                          02-Dec-2017 14:23                   -
	<a href="rhel-7-fast-datapath-rpms/">rhel-7-fast-datapath-rpms/</a>                         29-Apr-2017 19:48                   -
	<a href="rhel-7-server-extras-rpms/">rhel-7-server-extras-rpms/</a>                         29-Apr-2017 19:51                   -
	<a href="rhel-7-server-optional-rpms/">rhel-7-server-optional-rpms/</a>                       18-Aug-2017 21:13                   -
	<a href="rhel-7-server-ose-3.7-rpms/">rhel-7-server-ose-3.7-rpms/</a>                        01-Dec-2017 16:03                   -
	<a href="rhel-7-server-rpms/">rhel-7-server-rpms/</a>                                19-Aug-2017 06:02                   -
	</pre><hr></body>
	</html>

通过`yum list`命令检查YUM源可被正常访问，OpenShift相关的RPM包存在并版本无误。命令及输出如下：

	[root@yum ~]# yum list|grep atomic-openshift
	atomic-openshift.x86_64         3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-clients.x86_64 3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-clients-redistributable.x86_64
	atomic-openshift-cluster-capacity.x86_64
	atomic-openshift-descheduler.x86_64
	atomic-openshift-docker-excluder.noarch
	atomic-openshift-dockerregistry.x86_64
	atomic-openshift-excluder.noarch
	atomic-openshift-federation-services.x86_64
	atomic-openshift-master.x86_64  3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-node.x86_64    3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-node-problem-detector.x86_64
	atomic-openshift-pod.x86_64     3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-sdn-ovs.x86_64 3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-service-catalog.x86_64
	atomic-openshift-template-service-broker.x86_64
	atomic-openshift-tests.x86_64   3.7.9-1.git.0.7c71a2d.el7
	atomic-openshift-utils.noarch   3.7.9-1.git.7.eedd332.el7
	tuned-profiles-atomic-openshift-node.x86_64
		
## 3.2 搭建镜像仓库服务

本节在<font color="red">`registry.example.com`主机上安装镜像仓库服务</font>。请登录`registry.example.com`主机进行操作。

### 3.2.1 安装仓库服务

安装镜像仓库Docker Distribution。命令如下:
	
	[root@registry ~]# yum install -y docker-distribution
	
### 3.2.2 配置TLS
为了启用TLS协议传输，需要生成自签名证书。命令如下：

	[root@registry ~]# mkdir /etc/crts/ && cd /etc/crts
	[root@registry ~]# openssl req \
       -newkey rsa:2048 -nodes -keyout example.com.key \
       -x509 -days 365 -out example.com.crt -subj \
       "/C=CN/ST=GD/L=SZ/O=Global Security/OU=IT Department/CN=*.example.com"

编辑镜像仓库服务配置文件`/etc/docker-distribution/registry/config.yml`。确认http的配置如下：

	http:
       addr: :443
       tls:
           certificate: /etc/crts/example.com.crt
           key: /etc/crts/example.com.key
> 提示：YAML文件对格式和空格非常敏感，编辑配置时请注意。          

修改完毕后，刷新systemd配置。命令如下：
	    
	[root@registry ~]# systemctl daemon-reload

重启Docker Distribution服务。命令如下：

	[root@registry ~]# systemctl restart docker-distribution
	[root@registry ~]# systemctl enable docker-distribution
	
服务成功启动后，可以看到443端口已经被Docker Distribution监听。

	[root@registry crts]# netstat -tlnp|grep registry
	tcp6       0      0 :::443                  :::*                    LISTEN      104156/registry 

配置信任自签名证书。并重启Docker服务
	
	[root@registry ~]# cp /etc/crts/example.com.crt /etc/pki/ca-trust/source/anchors/
	[root@registry ~]# update-ca-trust extract
	
安装Docker服务。命令如下：

	[root@registry ~]# yum install -y docker python-setuptools
	[root@registry ~]# systemctl enable docker 
	[root@registry ~]# systemctl restart docker 

### 3.2.3 导入容器镜像

假设已经准备好了OpenShift容器镜像的离线安装包`ocp-3.7.9-images.tar.gz`，首先需要将镜像导入到registry.exmaple.com主机本地。

	[root@registry ~]# docker load -i ocp-3.7.9-images.tar.gz
	
修改所有镜像的名称，指向`registry.exmaple.com`。以镜像`ose-pod`为例子，命令示例如下：

	[root@registry ~]# docker tag registry.access.redhat.com/openshift3/ose-pod:v3.7.9 registry.example.com/openshift3/ose-pod:v3.7.9 
	
>	由于涉及的镜像众多，可以使用如下命令批量修改镜像名称。
>	
>		docker images |grep "redhat.com"|awk '{print "docker tag "$3" "$1":"$2}'| \
>		sed -e s/access.redhat.com/example.com/| \
>		xargs -i bash -c "{}"
	
将修改好名称的镜像推送至目标镜像仓库。以镜像`ose-pod`为例子，命令示例如下：

	[root@registry ~]# docker push registry.example.com/openshift3/ose-pod:v3.7.9 

>	由于涉及的镜像众多，可以使用如下命令批量修改推送镜像。
>
>		docker images |grep "example.com"| \
>		awk '{print "docker push "$1":"$2}'| \
>		xargs -i bash -c "{}"

# 4 准备主机

<font color="red">请在所有Master及Node节点上执行本小节的配置。</font>

## 4.1 配置YUM源
配置YUM源以便后续安装所需要的RPM软件包。创建文件`/etc/yum.repos.d/ocp.repo`。内容如下：

	[rhel-7-server-rpms]
	name=rhel-7-server-rpms
	baseurl=http://yum.example.com/repos/rhel-7-server-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-server-extras-rpms]
	name=rhel-7-server-extras-rpms
	baseurl=http://yum.example.com/repos/rhel-7-server-extras-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-fast-datapath-rpms]
	name=rhel-7-fast-datapath-rpms
	baseurl=http://yum.example.com/repos/rhel-7-fast-datapath-rpms
	enabled=1
	gpgcheck=0
	[rhel-7-server-ose-3.7-rpms]
	name=rhel-7-server-ose-3.7-rpms
	baseurl=http://yum.example.com/repos/rhel-7-server-ose-3.7-rpms
	enabled=1
	gpgcheck=0
	
删除Node2节点上的配置文件`/etc/yum.repos.d/local.repo`。

	rm -f /etc/yum.repos.d/local.repo
	
配置完毕后，更新并检查YUM仓库信息。执行如下命令：

	[root@所有节点 ~]# yum clean all && yum repolist
	
配置成功后可以见到相关的仓库的信息。示例输出如下：

	repo id                                                            repo name                                                           status
	rhel-7-fast-datapath-rpms                                          rhel-7-fast-datapath-rpms                                               16
	rhel-7-server-extras-rpms                                          rhel-7-server-extras-rpms                                              102
	rhel-7-server-ose-3.7-rpms                                         rhel-7-server-ose-3.7-rpms                                          497+10
	rhel-7-server-rpms                                                 rhel-7-server-rpms                                                   5,161
	repolist: 5,776

## 4.2 安装基础软件包

安装OpenShift运行及管理所需的基础软件包。命令如下：

	[root@所有节点 ~]# yum install -y wget git net-tools bind-utils iptables-services \
	bridge-utils bash-completion kexec-tools sos psacct vim lrzsz python-setuptools
	
更新系统软件包。命令如下：

	[root@所有节点 ~]# yum update -y
	
更新完成后重启主机。命令如下：

	[root@所有节点 ~]# reboot
	
##4.3 防火墙配置

测试环境，为了简化，直接放行数据包。命令如下：

	[root@所有节点 ~]# sed -i -e s/'-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT'/'-A INPUT -j ACCEPT'/g /etc/sysconfig/iptables
	[root@所有节点 ~]# systemctl restart iptables
	
## 4.4 安装Docker

安装所需版本的Docker。命令如下：

	[root@所有节点 ~]# yum install -y docker-1.12.6
		
启动并配置Docker服务开机启动。命令如下：

	[root@所有节点 ~]# systemctl start docker
	[root@所有节点 ~]# systemctl enable docker
	
## 4.5 配置Docker	
修改Docker配置文件`/etc/sysconfig/docker`，确保OPTIONS变量配置了如下参数。

	OPTIONS='--insecure-registry=172.30.0.0/16 --selinux-enabled --log-opt max-size=1M --log-opt max-file=3'	
修改ADD_REGISTRY变量。将如下内容：

	ADD_REGISTRY='--add-registry registry.access.redhat.com'
	
修改为：

	ADD_REGISTRY='--add-registry registry.example.com'
	
配置镜像仓库证书信任。命令如下：

	[root@所有节点 ~]# scp registry.example.com:/etc/crts/example.com.crt /etc/pki/ca-trust/source/anchors/
	[root@所有节点 ~]# update-ca-trust extract
	
配置修改完毕后，重启Docker服务。

	 [root@所有节点 ~]# systemctl restart docker
	 
配置完毕后，确认镜像可以正常拉取。

	[root@所有节点 ~]# docker pull registry.example.com/openshift3/ose-pod:v3.7.9
 
# 5 安装前配置

本文以Master节点作为安装的堡垒机。本小节所有的操作均在Master节点上执行。

## 5.1 配置SSH互信

在`Master节点`上生成SSH Key。

	[root@master ~]# ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

配置Master节点到各个Node节点及Master节点的互信。
	
	[root@master ~]# ssh-copy-id master.example.com
	[root@master ~]# ssh-copy-id node1.example.com
	[root@master ~]# ssh-copy-id node2.example.com
	[root@master ~]# ssh-copy-id nfs.example.com
	
## 5.2 安装Ansible

OpenShfit的安装通过Ansible自动完成，因此需要先安装Ansible及OpenShfit相关的Ansible Playbook。

	[root@master ~]# yum install -y openshift-ansible

## 5.3 设置安装配置

编辑`/etc/ansible/hosts`文件。将文件内容替换为如下内容：

	[OSEv3:children]
	masters
	nodes
	etcd
	nfs
	
	[OSEv3:vars]
	ansible_ssh_user=root
	#ansible_become=true
	openshift_deployment_type=openshift-enterprise
	openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
	openshift_disable_check=docker_image_availability,docker_storage,memory_availability
	
	oreg_url=registry.example.com/openshift3/ose-${component}:${version}
	openshift_examples_modify_imagestreams=true
	
	openshift_clock_enabled=true
	
	openshift_service_catalog_image_prefix=registry.example.com/openshift3/ose-
	openshift_service_catalog_image_version=v3.7
	
	openshift_hosted_router_replicas=1
	openshift_hosted_router_selector='router=yes'
	openshift_master_default_subdomain=apps.example.com
	
	openshift_hosted_etcd_storage_kind=nfs
	openshift_metrics_install_metrics=true
	openshift_metrics_hawkular_hostname=hawkular-metrics.apps.example.com
	openshift_metrics_cassandra_storage_type=emptydir
	openshift_metrics_image_prefix=registry.example.com/openshift3/
	
	#openshift_logging_install_logging=true
	#openshift_logging_image_prefix=registry.example.com/openshift3/
	
	# host group for masters
	[masters]
	master.example.com 
		
	# host group for etcd
	[etcd]
	master.example.com
	
	# host group for nodes, includes region info
	[nodes]
	master.example.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
	node1.example.com openshift_node_labels="{'region': 'infra','router': 'yes', 'zone': 'default'}" openshift_schedulable=true
	node2.example.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}" openshift_schedulable=true
	
	[nfs]
	nfs.example.com
	

# 6 执行安装

安装前再次检查所有的节点已经配置并就绪。在Master节点上执行如下命令：

	[root@master ~]# ansible -m shell -a 'hostname' nodes
	[root@master ~]# ansible -m shell -a 'docker pull registry.example.com/openshift3/ose-pod:v3.7.9' nodes
	[root@master ~]# ansible -m shell -a 'yum repolist' nodes

执行安装。命令如下：
     
	[root@master ~]# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml
	
当安装显示如下信息时，说明安装成功结束。

	PLAY RECAP *********************************************************************************************************************************************************************
	localhost                  : ok=13   changed=0    unreachable=0    failed=0   
	master.example.com         : ok=759  changed=178  unreachable=0    failed=0   
	nfs.example.com            : ok=63   changed=3    unreachable=0    failed=0   
	node1.example.com          : ok=174  changed=13   unreachable=0    failed=0   
	node2.example.com          : ok=174  changed=13   unreachable=0    failed=0   
	
	
	INSTALLER STATUS ***************************************************************************************************************************************************************
	Initialization             : Complete
	Health Check               : Complete
	etcd Install               : Complete
	NFS Install                : Complete
	Master Install             : Complete
	Master Additional Install  : Complete
	Node Install               : Complete
	Hosted Install             : Complete
	Metrics Install            : Complete
	Service Catalog Install    : Complete


查看集群节点，确认集群正常。命令及示例输出如下：

	[root@master ~]# oc get node
	NAME                 STATUS                     AGE       VERSION
	master.example.com   Ready,SchedulingDisabled   1h        v1.7.6+a08f5eeb62
	node1.example.com    Ready                      1h        v1.7.6+a08f5eeb62
	node2.example.com    Ready                      1h        v1.7.6+a08f5eeb62

查看集群容器，确认各个组件状态正常。命令及示例输出如下：
	
	[root@master ~]# oc get pod --all-namespaces
	NAMESPACE                           NAME                         READY     STATUS    RESTARTS   AGE
	default                             docker-registry-1-ffp9q      1/1       Running   0          52m
	default                             registry-console-1-qbp7c     1/1       Running   0          52m
	default                             router-2-mtckd               1/1       Running   0          54m
	kube-service-catalog                apiserver-plxqf              1/1       Running   0          46m
	kube-service-catalog                controller-manager-mbgfk     1/1       Running   0          46m
	openshift-ansible-service-broker    asb-1-h7x45                  1/1       Running   0          42m
	openshift-ansible-service-broker    asb-etcd-1-zq7gb             1/1       Running   0          42m
	openshift-infra                     hawkular-cassandra-1-pbfcm   1/1       Running   0          49m
	openshift-infra                     hawkular-metrics-pn5dv       1/1       Running   0          49m
	openshift-infra                     heapster-nkxmd               1/1       Running   0          49m
	openshift-template-service-broker   apiserver-4n8jk              1/1       Running   0          41m
	openshift-template-service-broker   apiserver-grgjx              1/1       Running   0          41m
	openshift-template-service-broker   apiserver-vzvnk              1/1       Running   0          41m


# 7 安装后配置


## 7.1 创建用户

创建用户`dev`，密码为`welcome1`。命令如下：

	[root@master ~]# htpasswd -cb /etc/origin/master/htpasswd dev welcome1
	Adding password for user dev

通过浏览器访问`https://master.example.com:8443`即可登录OpenShift的Web Console。

## 7.2 域名解析（可选）

集群外部访问OpenShift的应用需要使用`*.apps.example.com`域名。为了测试方便，可以配置`*.apps.example.com`的泛域名解析。

编辑配置`Master`主机的文件`/etc/dnsmasq.d/openshift-cluster.conf`。文件内容如下：
 
	local=/example.com/
	address=/.apps.example.com/192.168.172.202

重启dnsmasq服务。

	[root@master ~]# systemctl restart dnsmasq
	
此时Master节点上的dnsmasq服务将可以解析*.apps.exmaple.com的域名请求。在有需要的主机上将域名解析服务器指向Master即可。在Linux上，编辑各个节点的`/etc/resolv.conf`文件。添加如下信息：

	nameserver 192.168.172.201
	
Windows的桌面请在网络设置界面修改域名解析服务器的配置。
	
配置完毕后，测试一个随机域名是否可以被正常解析。

	[root@master ~]# ping abc.apps.example.com
	PING abc.apps.example.com (192.168.172.202) 56(84) bytes of data.
	64 bytes from node1.example.com (192.168.172.202): icmp_seq=1 ttl=64 time=0.320 ms
	64 bytes from node1.example.com (192.168.172.202): icmp_seq=2 ttl=64 time=0.216 ms
	64 bytes from node1.example.com (192.168.172.202): icmp_seq=3 ttl=64 time=0.560 ms


## 7.3 创建持久化资源池（可选）

为了方便测试与使用，可以提前创建一批PV。命令如下：

	cat >/tmp/pv.temp.yaml<<EOF
	apiVersion: v1
	kind: PersistentVolume
	metadata:
	  name: \$pvi
	spec:
	  capacity:
	    storage: 5Gi
	  accessModes:
	  - ReadWriteOnce
	  - ReadWriteMany
	  nfs:
	    path: /exports/\$pvi
	    server: nfs.example.com
	  persistentVolumeReclaimPolicy: Recycle
	EOF
	  
	for i in $(seq 1 20); do mkdir -p /exports/pv$i ; done;
	chown -R nfsnobody. /exports/
	ls /exports/|xargs -i echo /exports/{} *(rw,all_squash) >> /etc/exports;
	systemctl restart rpcbind nfs-server
	exportfs -r
	showmount -e
	for i in $(seq 1 20); do export pvi=pv$i; cat /tmp/pv.temp.yaml|envsubst|oc create -f - ;done;

创建完毕后可见创建好的PV。

	[root@master ~]# oc get pv
	NAME          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                                   STORAGECLASS   REASON    AGE
	etcd-volume   1Gi        RWO           Retain          Bound       openshift-ansible-service-broker/etcd                            1h
	pv1           5Gi        RWO,RWX       Recycle         Available                                                                    52s
	pv10          5Gi        RWO,RWX       Recycle         Available                                                                    49s
	pv11          5Gi        RWO,RWX       Recycle         Available                                                                    28s
	pv12          5Gi        RWO,RWX       Recycle         Available                                                                    27s
	pv13          5Gi        RWO,RWX       Recycle         Available                                                                    27s
	pv14          5Gi        RWO,RWX       Recycle         Available                                                                    27s
	pv15          5Gi        RWO,RWX       Recycle         Available                                                                    26s
	pv16          5Gi        RWO,RWX       Recycle         Available                                                                    26s
	pv17          5Gi        RWO,RWX       Recycle         Available                                                                    26s
	pv18          5Gi        RWO,RWX       Recycle         Available                                                                    25s
	pv19          5Gi        RWO,RWX       Recycle         Available                                                                    25s
	pv2           5Gi        RWO,RWX       Recycle         Available                                                                    52s
	pv20          5Gi        RWO,RWX       Recycle         Available                                                                    24s
	pv3           5Gi        RWO,RWX       Recycle         Available                                                                    51s
	pv4           5Gi        RWO,RWX       Recycle         Available                                                                    51s
	pv5           5Gi        RWO,RWX       Recycle         Available                                                                    51s
	pv6           5Gi        RWO,RWX       Recycle         Available                                                                    50s
	pv7           5Gi        RWO,RWX       Recycle         Available                                                                    50s
	pv8           5Gi        RWO,RWX       Recycle         Available                                                                    50s
	pv9           5Gi        RWO,RWX       Recycle         Available                                                                    49s


## 7.4 日志聚合（可选）

如需要日志聚合组件，可将`/etc/ansible/hsots`文件的下面两行内容的注释去除。

	openshift_logging_install_logging=true
	openshift_logging_image_prefix=registry.example.com/openshift3/
	
然后执行Ansible Playbook。命令如下：

	[root@master ~]# ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-logging.yml

成功安装完毕后，可以见到Logging组件相关的容器实例。
	
	[root@master ~]# oc get pod -n logging
	NAME                                       READY     STATUS    RESTARTS   AGE
	logging-curator-1-8hzsl                    1/1       Running   4          25m
	logging-es-data-master-fu18dd1u-2-gc9cq    2/2       Running   0          8m
	logging-fluentd-flcjf                      1/1       Running   0          25m
	logging-fluentd-hmfkh                      1/1       Running   0          25m
	logging-kibana-4-xk56f                     2/2       Running   0          1m


Kibana及Elastic Search默认要求的内存比较高，如遇到容器出现Error状态，可能是内存不足导致。此时可以考虑增加Node节点的内存。通过以下命令可以查看具体错误的信息。

	[root@master ~]# oc describe pod <pod的名字>

# 8 验证安装

以前文创建的用户`dev`登录系统。

	[root@master ~]# oc login -u dev 
	Authentication required for https://master.example.com:8443 (openshift)
	Username: dev
	Password: 

> 提示：通过如下的命令可以再次登录成集群管理员。
>	
>		oc login -u system:admin

创建项目。

	[root@master ~]# oc new-project demo

部署容器应用。

	[root@master ~]# oc new-app openshift/hello-openshift
	
容器成功启动。
	
	[root@master ~]# oc get pod -o wide
	NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
	hello-openshift-1-4hjrt   1/1       Running   0          2m        10.128.0.54   node2.example.com

通过`curl`访问容器服务。

	[root@master ~]# curl 10.128.0.54:8080
	Hello OpenShift!

以上操作也可以在OpenShift的Web Console界面完成。此时登录Web Console，可以亦可见到前文部署的容器。

为应用创建Route以便外部可以通过域名访问。

	[root@master ~]# oc expose svc hello-openshift
	route "hello-openshift" exposed

查看生产的Route信息。

	[root@master ~]# oc get route
	NAME              HOST/PORT                               PATH      SERVICES          PORT       TERMINATION   WILDCARD
	hello-openshift   hello-openshift-demo.apps.example.com             hello-openshift   8080-tcp                 None
		
通过域名访问服务。

	[root@master ~]# curl hello-openshift-demo.apps.example.com
	Hello OpenShift!
	
> 请确保域名`hello-openshift-demo.apps.example.com`将被解析到Router所在的Node的节点的IP。可以通过修改`/etc/hosts`实现，也可以通过配置dnsmasq或bind等服务实现。请参考章节7.2。

# 附录
	
## 附录A 制作离线RPM包

在可联网的RHEL主机上注册红帽订阅，关联到相应的YUM仓库。命令如下：

	# subscription-manager register
	# subscription-manager refresh
	# subscription-manager list --available --matches '*OpenShift*'
	# subscription-manager attach --pool=<pool_id>
	# subscription-manager repos --disable="*"
	# subscription-manager repos \
	    --enable="rhel-7-server-rpms" \
	    --enable="rhel-7-server-extras-rpms" \
	    --enable="rhel-7-fast-datapath-rpms" \
	    --enable="rhel-7-server-ose-3.7-rpms"

同步所需的仓库至本地。命令如下：

	# yum -y install yum-utils createrepo docker git
	# mkdir -p /opt/repos/
	# for repo in \
	rhel-7-server-rpms \
	rhel-7-server-extras-rpms \
	rhel-7-fast-datapath-rpms \
	rhel-7-server-ose-3.7-rpms
	do
	  reposync --gpgcheck -lm --repoid=${repo} --download_path=/opt/repos/
	  createrepo -v /opt/repos/${repo} -o /opt/repos/${repo}
	done

压缩/opt/repos目录。压缩完毕后的压缩包即可用于离线环境的安装。

	# cd /opt/repos/
	# tar zcvf rhel-7-server-rpms.tar.gz rhel-7-server-rpms 
	# tar zcvf rhel-7-server-extras-rpms.tar.gz rhel-7-server-extras-rpms
	# tar zcvf rhel-7-fast-datapath-rpms.tar.gz rhel-7-fast-datapath-rpms
	# tar zcvf rhel-7-server-ose-3.7-rpms.tar.gz rhel-7-server-ose-3.7-rpms

	
## 附录B 制作离线镜像包

在可联网的RHEL主机上下载如下容器镜像。
	 
	docker pull registry.access.redhat.com/openshift3/logging-auth-proxy:v3.7
	docker pull registry.access.redhat.com/openshift3/logging-curator:v3.7
	docker pull registry.access.redhat.com/openshift3/logging-elasticsearch:v3.7
	docker pull registry.access.redhat.com/openshift3/logging-fluentd:v3.7
	docker pull registry.access.redhat.com/openshift3/logging-kibana:v3.7
	docker pull registry.access.redhat.com/openshift3/metrics-cassandra:v3.7
	docker pull registry.access.redhat.com/openshift3/metrics-hawkular-metrics:v3.7
	docker pull registry.access.redhat.com/openshift3/metrics-heapster:v3.7
	docker pull registry.access.redhat.com/openshift3/oauth-proxy:v3.7
	docker pull registry.access.redhat.com/openshift3/ose-ansible-service-broker:v3.7
	docker pull registry.access.redhat.com/openshift3/ose-deployer:v3.7.9
	docker pull registry.access.redhat.com/openshift3/ose-docker-registry:v3.7.9
	docker pull registry.access.redhat.com/openshift3/ose-haproxy-router:v3.7.9
	docker pull registry.access.redhat.com/openshift3/ose-pod:v3.7.9
	docker pull registry.access.redhat.com/openshift3/ose-service-catalog:v3.7
	docker pull registry.access.redhat.com/openshift3/ose-sti-builder:v3.7.9
	docker pull registry.access.redhat.com/openshift3/ose:v3.7
	docker pull registry.access.redhat.com/openshift3/registry-console:v3.7
	docker pull registry.access.redhat.com/rhel7/etcd:latest
	docker pull registry.access.redhat.com/rhscl/php-70-rhel7:latest
	docker pull registry.access.redhat.com/openshift3/jenkins-2-rhel:latest
	docker pull registry.access.redhat.com/openshift3/jenkins-slave-maven-rhel7:latest
	docker pull registry.access.redhat.com/rhscl/mysql-57-rhel7:latest
	docker pull docker.io/openshift/hello-openshift:latest
	docker pull docker.io/gogs/gogs:latest
	docker pull docker.io/sonatype/nexus3:latest
	
将下载好的镜像保存成tar包。保存的过程需要消耗较多的磁盘空间及比较耗时。可以考虑分组保存成多个tar包。

	docker save -o ocp-3.7-core-images.tar \
	registry.access.redhat.com/rhel7/etcd:latest \
	registry.access.redhat.com/rhscl/php-70-rhel7:latest \
	registry.access.redhat.com/openshift3/oauth-proxy:v3.7 \
	registry.access.redhat.com/openshift3/ose-service-catalog:v3.7 \
	registry.access.redhat.com/openshift3/ose-ansible-service-broker:v3.7 \
	registry.access.redhat.com/openshift3/registry-console:v3.7 \
	registry.access.redhat.com/openshift3/ose-sti-builder:v3.7.9 \
	registry.access.redhat.com/openshift3/ose-haproxy-router:v3.7.9 \
	registry.access.redhat.com/openshift3/ose-deployer:v3.7.9 \
	registry.access.redhat.com/openshift3/ose:v3.7 \
	registry.access.redhat.com/openshift3/ose-docker-registry:v3.7.9 \
	registry.access.redhat.com/openshift3/ose-pod:v3.7.9 
	
	docker save -o ocp-3.7-metrics-logging-images.tar \
	registry.access.redhat.com/openshift3/logging-kibana:v3.7 \
	registry.access.redhat.com/openshift3/metrics-hawkular-metrics:v3.7 \
	registry.access.redhat.com/openshift3/metrics-heapster:v3.7 \
	registry.access.redhat.com/openshift3/metrics-cassandra:v3.7 \
	registry.access.redhat.com/openshift3/logging-fluentd:v3.7 \
	registry.access.redhat.com/openshift3/logging-elasticsearch:v3.7 \
	registry.access.redhat.com/openshift3/logging-auth-proxy:v3.7 \
	registry.access.redhat.com/openshift3/logging-curator:v3.7 
	
	docker save -o ocp-3.7-extra-images.tar \
	registry.access.redhat.com/openshift3/jenkins-2-rhel7:latest \
	registry.access.redhat.com/openshift3/jenkins-slave-maven-rhel7:latest \
	registry.access.redhat.com/rhscl/mysql-57-rhel7:latest \
	docker.io/openshift/hello-openshift:latest \
	docker.io/gogs/gogs:latest \
	docker.io/sonatype/nexus3:latest  

为了节省空间，对`docker save`生成的tar包进行压缩。

	gzip ocp*images.tar
	
压缩完成后将生成tar.gz文件。将文件拷贝到其他机器上，通过命令`docker load -i <名字>.tar.gz`即可完成导入。如有其它需要离线的镜像，也可通过上述描述的步骤准备。   
   
## 附录C 安装常见问题

遇到无法排除的错误，可以尝试卸载OpenShift集群再次安装。命令如下：

	ansible-playbook  /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml;

# 参考材料
- https://docs.openshift.com/container-platform/3.7/install_config/install/advanced_install.html
- https://docs.openshift.com/container-platform/3.7/install_config/install/disconnected_install.html
