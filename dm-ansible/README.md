# DM Ansible 部署方案

## 概述

Ansible 是一款自动化运维工具，DM-Ansible 是 PingCAP 基于 Ansible playbook 功能编写的集群部署工具。使用 TiDB-Ansible 可以快速部署一个完整的 DM 集群。

本部署工具可以通过配置文件设置集群拓扑，完成以下各项运维工作：

- 初始化操作系统参数
- 部署 DM 集群组件（包括 dm-master、dm-worker、dmctl）
- 部署监控组件（包括 prometheus、grafana、alertmanager）
- 启动集群
- 关闭集群
- 升级组件版本

## 准备机器

1.  部署目标机器若干

    - 推荐安装 CentOS 7.3 及以上版本 Linux 操作系统，x86_64 架构 (amd64)。
    - 机器之间内网互通。

2.  部署中控机一台:

    - 中控机可以是部署目标机器中的某一台。
    - 推荐安装 CentOS 7.3 及以上版本 Linux 操作系统（默认包含 Python 2.7）。
    - 该机器需开放外网访问，用于下载 DM 及相关软件安装包。

## 在中控机上安装系统依赖包

以 `root` 用户登录中控机

如果中控机是 CentOS 7 系统，执行以下命令：

```
# yum -y install epel-release git curl sshpass
# yum -y install python-pip
```

如果是中控机是 Ubuntu 系统，执行以下命令：

```
# apt-get -y install git curl sshpass python-pip
```

## 在中控机上创建 tidb 用户，并生成 ssh key

以 `root` 用户登录中控机，执行以下命令

创建 `tidb` 用户

```
# useradd -m -d /home/tidb tidb
```

设置 `tidb` 用户密码

```
# passwd tidb
```

生成 ssh key: 执行 `su` 命令从 `root` 用户切换到 `tidb` 用户下，创建 `tidb` 用户 ssh key， 提示 `Enter passphrase` 时直接回车即可。执行成功后，ssh 私钥文件为 `/home/tidb/.ssh/id_rsa`， ssh 公钥文件为 `/home/tidb/.ssh/id_rsa.pub`。

```
# su - tidb
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/tidb/.ssh/id_rsa):
Created directory '/home/tidb/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/tidb/.ssh/id_rsa.
Your public key has been saved in /home/tidb/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:eIBykszR1KyECA/h0d7PRKz4fhAeli7IrVphhte7/So tidb@172.16.10.49
The key's randomart image is:
+---[RSA 2048]----+
|=+o+.o.          |
|o=o+o.oo         |
| .O.=.=          |
| . B.B +         |
|o B * B S        |
| * + * +         |
|  o + .          |
| o  E+ .         |
|o   ..+o.        |
+----[SHA256]-----+
```

## 在中控机器上下载 DM-Ansible

以 `tidb` 用户登录中控机并进入 `/home/tidb` 目录。

```
$ wget http://download.pingcap.org/dm-ansible.tar.gz
```

## 在中控机器上安装 Ansible 及其依赖

以 `tidb` 用户登录中控机，请务必按以下方式通过 pip 安装 Ansible 及其相关依赖的指定版本，否则会有兼容问题。安装完成后，可通过 `ansible --version` 查看 Ansible 版本。目前 DM-Ansible 兼容 Ansible 2.5 版本及以上版本，Ansible 及相关依赖版本记录在 `tidb-ansible/requirements.txt` 文件中。

  ```bash
  $ cd /home/tidb/tidb-ansible
  $ sudo pip install -r ./requirements.txt
  $ ansible --version
    ansible 2.5.0
  ```

## 在中控机上配置部署机器 ssh 互信及 sudo 规则

以 `tidb` 用户登录中控机，将你的部署目标机器 IP 添加到 `hosts.ini` 文件 `[servers]` 区块下。

```
$ cd /home/tidb/tidb-ansible
$ vi hosts.ini
[servers]
172.16.10.71
172.16.10.72
172.16.10.73

[all:vars]
username = tidb
```

执行以下命令，按提示输入部署目标机器 `root` 用户密码。该步骤将在部署目标机器上创建 `tidb` 用户，并配置 sudo 规则，配置中控机与部署目标机器之间的 ssh 互信。

```
$ ansible-playbook -i hosts.ini create_users.yml -u root -k
```

## 分配机器资源，编辑 inventory.ini 文件

以 `tidb` 用户登录中控机，`inventory.ini` 文件路径为 `/home/tidb/dm-ansible/inventory.ini`。

> **注：** 请使用内网 IP 来部署集群，如果部署目标机器 SSH 端口非默认 22 端口，需添加 `ansible_port` 变量，如：
> `dm-worker1 ansible_host=172.16.10.72 ansible_port=5555`

| Name | Host IP | Services |
| ---- | ------- | -------- |
| node1 | 172.16.10.71 | dm-master, prometheus, grafana, alertmanager |
| node2 | 172.16.10.72 | dm-worker1 |
| node3 | 172.16.10.73 | dm-worker2 |

```ini
## DM modules
[dm_master_servers]
dm-master ansible_host=172.16.10.71

[dm_worker_servers]
dm-worker1 ansible_host=172.16.10.72 server_id=101 mysql_host=172.16.10.72 mysql_user=root mysql_password='KYMz623KG+6uspkXMJrU+0BYknT8xVU='

dm-worker2 ansible_host=172.16.10.73 server_id=102 mysql_host=172.16.10.73 mysql_user=root mysql_password='KYMz623KG+6uspkXMJrU+0BYknT8xVU='

## Monitoring modules
[prometheus_servers]
prometheus ansible_host=172.16.10.71

[grafana_servers]
grafana ansible_host=172.16.10.71

[alertmanager_servers]
alertmanager ansible_host=172.16.10.71
```

### inventory.ini 变量调整

#### 部署目录调整

部署目录通过 `deploy_dir` 变量控制，默认全局变量已设置为 `/home/tidb/deploy`，对所有服务生效。如数据盘挂载目录为 `/data1`，可设置为 `/data1/deploy`，样例如下:

```
## Global variables
[all:vars]
deploy_dir = /data1/dm
```

如为某一服务单独设置部署目录，可在配置服务主机列表时配置主机变量，以 dm-master 节点为例，其他服务类推，请务必添加第一列别名，以免服务混布时混淆。

```
dm-master ansible_host=172.16.10.71 deploy_dir=/data1/deploy
```

#### 其他变量调整

> **注：** 以下控制变量开启请使用首字母大写 `True`，关闭请使用首字母大写 `False`。

| 变量            | 含义                                                        |
| --------------- | ---------------------------------------------------------- |
| cluster_name | 集群名称，可调整 |
| dm_version | DM 版本，默认已配置 |
| grafana_admin_user | Grafana 管理员帐号用户名，默认为 admin |
| grafana_admin_password | Grafana 管理员帐号密码，默认为 admin，用于 Ansible 导入 Dashboard，如后期通过 grafana web 修改了密码，请更新此变量 |

## 部署任务

> ansible-playbook 执行 Playbook 时默认并发为 5，部署目标机器较多时可添加 -f 参数指定并发，如 `ansible-playbook deploy.yml -f 10`

1.  确认 `dm-ansible/inventory.ini` 文件中 `ansible_user = tidb`，本例使用 `tidb` 用户作为服务运行用户，配置如下：

    > `ansible_user` 不要设置成 `root` 用户，`dm-ansbile` 限制了服务以普通用户运行。

    执行以下命令如果所有 server 返回 `tidb` 表示 ssh 互信配置成功。

    ```
    ansible -i inventory.ini all -m shell -a 'whoami'
    ```

    执行以下命令如果所有 server 返回 `root` 表示 `tidb` 用户 sudo 免密码配置成功。

    ```
    ansible -i inventory.ini all -m shell -a 'whoami' -b
    ```

2.  执行 `local_prepare.yml` playbook，联网下载 DM 及监控组件安装包到中控机：

    ```
    ansible-playbook local_prepare.yml
    ```

3.  初始化系统环境，修改内核参数

    ```
    ansible-playbook bootstrap.yml
    ```

4.  修改内核参数，并部署 DM 集群组件及监控组件

    ```
    ansible-playbook deploy.yml
    ```

5.  启动 DM 集群

    ```
    ansible-playbook start.yml
    ```

## 常见运维操作

### 启动集群

此操作会按顺序启动整个 DM 集群所有组件（包括 包括 dm-master、dm-worker、dmctl 和监控组件）。

```
$ ansible-playbook start.yml
```

### 关闭集群

此操作会按顺序关闭整个 TiDB 集群所有组件（（包括 包括 dm-master、dm-worker、dmctl 和监控组件）。

```
$ ansible-playbook stop.yml
```

## 常见部署问题

### 如何自定义端口

修改 `inventory.ini` 文件，在相应服务 IP 后添加以下主机变量即可：

```
dm-master ansible_host=172.16.10.71 dm_master_port=12080 dm_master_status_port=12081
```

| 组件 | 端口变量 | 默认端口 | 说明 |
| :-- | :-- | :-- | :-- |
| dm-master | dm_master_port | 11080  | dm-master 服务通信端口 |
| dm-master | dm_master_status_port | 11081  | dm-master 状态端口 |
| dm-worker | dm_worker_port | 10081  | dm-worker 服务通信端口 |
| dm-worker | dm_master_status_port | 10082  | dm-worker 状态端口 |
| Prometheus | prometheus_port | 9090 | Prometheus 服务通信端口 |
| Grafana | grafana_port |  3000 | Web 监控服务对外服务和客户端(浏览器)访问端口 |
| Alertmanager | alertmanager_port |  9093 | Alertmanager 服务通信端口 |
