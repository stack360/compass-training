compass4nfv的ansible部分及OpenStack适配
======================================

1. 简介
-----------
- 代码来源：https://github.com/opnfv/compass4nfv
- 讲解的目录为：compass4nfv/deploy/adapters/ansible
- 目录下面分为一些openstack不同版本的基类，默认的roles对应的是newton的版本
- 对compass4nfv的ansible目录部分进行讲解，和进行版本升级适配可能涉及的工作


2. 适配简介
--------------
- compass4nfv的coloroda版本支持几个版本的OpenStack的安装
- 准备在o版本发布之前代码的master-branch只留下n版本的内容
- 版本适配的工作集中在ansible方面，因为cobbler安装操作系统的改动很少


3. 代码解析--playbook入口
-------------------------
- 在跑ansible的时候的入口是/openstack/HA-ansible-multinodes.yml
- 重要的网站：http://docs.ansible.com/ansible/list_of_all_modules.html
- 下面是源码和一些注释：

```
---
- hosts: all
  remote_user: root
  pre_tasks:                                #这些是最先执行的tasks，利用ssh打通各机器的连接，ansible默认使用ssh进行通信
    - name: make sure ssh dir exist
      file:                                 #module的名称，ansible有许多内置的module，详见http://docs.ansible.com/ansible/list_of_all_modules.html
        path: '{{ item.path }}'
        owner: '{{ item.owner }}'
        group: '{{ item.group }}'
        state: directory
        mode: 0755
      with_items:                           #上面的操作需要的参数
        - path: /root/.ssh
          owner: root
          group: root

    - name: write ssh config
      copy:
        content: "UserKnownHostsFile /dev/null\nStrictHostKeyChecking no"
        dest: '{{ item.dest }}'
        owner: '{{ item.owner }}'
        group: '{{ item.group }}'
        mode: 0600
      with_items:
        - dest: /root/.ssh/config
          owner: root
          group: root

    - name: generate ssh keys
      shell: if [ ! -f ~/.ssh/id_rsa.pub ]; \
             then ssh-keygen -q -t rsa -f ~/.ssh/id_rsa -N ""; \
             else echo "already gen ssh key!"; fi;

    - name: fetch ssh keys
      fetch:
        src: /root/.ssh/id_rsa.pub
        dest: /tmp/ssh-keys-{{ ansible_hostname }}
        flat: "yes"

    - authorized_key:
        user: root
        key: "{{ lookup('file', item) }}"
      with_fileglob:
        - /tmp/ssh-keys-*
  max_fail_percentage: 0                  #pre_tasks允许通过的失败率为0%，也就是任何task失败都会导致pre_tasks的失败
  roles:
    - common                              #运行common的role，接下来的安装基本都是这个模式，调用roles

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - setup-network                       #启动网络设置

- hosts: ha
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ha                                  #安装ha

- hosts: controller                       #先对controller中各组件进行安装
  remote_user: root
  max_fail_percentage: 0
  roles:                                  #对于存在一定依赖的组件，需要严格按照依赖关系安排安装顺序
    - memcached                           #对于不存在依赖关系的组件，安装关系可以对换
    - apache
    - database
    - mq
    - keystone                            #在controller的各组件中，keystone一定要放置在前面
    - nova-controller
    - neutron-controller
    - cinder-controller
    - glance
    - neutron-common
    - neutron-network
    - ceilometer_controller
    - dashboard
    - heat
    - aodh
    - congress

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - storage

- hosts: compute                        #compute节点涉及到的4个最主要的openstack组件
  remote_user: root
  max_fail_percentage: 0
  roles:
    - nova-compute
    - neutron-compute
    - cinder-volume
    - ceilometer_compute

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles: []
#    - moon

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - secgroup

- hosts: ceph_adm
  remote_user: root
  max_fail_percentage: 0
  roles: []
#    - ceph-deploy

- hosts: ceph                           #ceph的各种配置
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ceph-purge
    - ceph-config

- hosts: ceph_mon
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ceph-mon

- hosts: ceph_osd
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ceph-osd

- hosts: ceph
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ceph-openstack

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - monitor

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  tasks:
    - name: set bash to nova                    #切到nova进行操作
      user:                                     #以下ssh的设置可以参照pre_tasks
        name: nova
        shell: /bin/bash

    - name: make sure ssh dir exist
      file:
        path: '{{ item.path }}'
        owner: '{{ item.owner }}'
        group: '{{ item.group }}'
        state: directory
        mode: 0755
      with_items:
        - path: /var/lib/nova/.ssh
          owner: nova
          group: nova

    - name: copy ssh keys for nova
      shell: cp -rf /root/.ssh/id_rsa /var/lib/nova/.ssh;

    - name: write ssh config
      copy:
        content: "UserKnownHostsFile /dev/null\nStrictHostKeyChecking no"
        dest: '{{ item.dest }}'
        owner: '{{ item.owner }}'
        group: '{{ item.group }}'
        mode: 0600
      with_items:
        - dest: /var/lib/nova/.ssh/config
          owner: nova
          group: nova

    - authorized_key:
        user: nova
        key: "{{ lookup('file', item) }}"
      with_fileglob:
        - /tmp/ssh-keys-*

    - name: chown ssh file
      shell: chown -R nova:nova /var/lib/nova/.ssh;

- hosts: all                                   #odl和onos的一些设置
  remote_user: root
  max_fail_percentage: 0
  roles:
    - odl_cluster

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - onos_cluster

- hosts: all
  remote_user: root
  serial: 1
  max_fail_percentage: 0
  roles:
    - odl_cluster_neutron

- hosts: all
  remote_user: root
  max_fail_percentage: 0
  roles:
    - odl_cluster_post

- hosts: controller
  remote_user: root
  max_fail_percentage: 0
  roles:
    - ext-network

- hosts: controller                         #nano的设置，vm的规格的配置等，或者一些新增加的组件和功能都放置在这里
  remote_user: root
  max_fail_percentage: 0
  roles:
    - openstack-post

- hosts: controller                         #如果涉及到突然的断电情况，使用以下recovery进行恢复
  remote_user: root
  max_fail_percentage: 0
  roles:
    - boot-recovery

- hosts: controller
  remote_user: root
  max_fail_percentage: 0
  roles:
    - controller-recovery

- hosts: compute
  remote_user: root
  max_fail_percentage: 0
  roles:
    - compute-recovery
```

4. keystone
----------------------------------------
- 以keystone为例，对roles的安装运行方式进行解析
- 关于keystone token的解释见：http://liyuenan.com/
- keystone的在roles中的目录

```
|--roles
   |--keystone
      |--handlers                   #待用的一些task，需要在执行过程中被notify才会运行，否则不会运行
         |--main.yml
      |--tasks
         |--keystone_config.yml
         |--keystone_create.yml
         |--keystone_install.yml
         |--main.yml                #playbook的执行入口，调用以上三个文件，顺序为install-->config-->create
      |--templates                  #模板，采用jinja2进行渲染
         |--admin-openrc-v2.sh
         |--admin-openrc.sh
         |--demo-openrc.sh
         |--keystone.conf
         |--wsgi-keystone.conf.j2
      |--vars                       #运行需要的参数，为了方便在不同操作系统中的运行，这里除了main之外
         |--Debian.yml              #还分别写了适配于Ubuntu和RedHat的参数
         |--main.yml
         |--RedHat.yml
```

5. 手动部署
-----------
- 手动部署是了解OpenStack的一个比较好的途径，一般第一次手动部署的工作时间在一周左右
- 手动部署的官方文档在（以n版本为例）：http://docs.openstack.org/project-install-guide/newton/
- Yunnan根据自己的时间有一份手动部署的文档，见目录下`OpenStack+Mitaka+for+Ubuntu+16.04部署手册`
- 实际操作中请对照以上两份文档，因为环境可能不同，在实际操作中以上两份文档中的细节需要对照具体环境使用和更改


6. compass4nfv部署OpenStack的一些工具
-----------------------------------------------
- 代码源：https://github.com/Yuenannn/compass_tools
- 对其中的ping_test.sh进行解析

```
#!/bin/bash
##############################################################################
# File Name:   ping_test.sh
# Revision:    2.0
# Date:        2017-02-08
# Author:      Yuenan Li
# Email:       liyuenan93@icloud.com
# Blog:        liyuenan.com
# Description: Test launch instance on openstack
##############################################################################

# Run this script in a controller node.
set -ex

# source the admin credentials to gain access to admin-only CLI commands:
source /opt/admin-openrc.sh

# Download the source image:   
# 下载cirros镜像
if [[ ! -e cirros-0.3.3-x86_64-disk.img ]]; then
    wget 10.1.0.12/image/cirros-0.3.3-x86_64-disk.img
fi

# Upload the image to the Image service using the QCOW2 disk format, bare container format:
#将镜像上传至glance也就是OpenStack的镜像服务
if [[ ! $(glance image-list | grep cirros) ]]; then
    glance image-create --name "cirros" \
        --file cirros-0.3.3-x86_64-disk.img  \
        --disk-format qcow2 --container-format bare
fi

# Add rules to the default security group:
# 通信协议及安全设置
if [[ ! $(nova secgroup-list-rules default | grep icmp) ]]; then
    nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
fi
if [[ ! $(nova secgroup-list-rules default | grep tcp) ]]; then
    nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
fi

# Create the net and a subnet on the network:
# 创建子网
if [[ ! $(neutron net-list | grep demo-net) ]]; then
    neutron net-create demo-net
fi
if [[ ! $(neutron subnet-list | grep demo-subnet) ]]; then
    neutron subnet-create demo-net 10.10.10.0/24 --name demo-subnet --gateway 10.10.10.1
fi

# Create the router, add the demo-net network subnet and set a gateway on the ext-net network on the router:
# 给已经创建的子网demo-net创建路由
if [[ ! $(neutron router-list | grep demo-router) ]]; then
    neutron router-create demo-router
    neutron router-interface-add demo-router demo-subnet
    neutron router-gateway-set demo-router ext-net
fi

# Create m1.test flavor
if [[ ! $(openstack flavor list | grep m1.test) ]]; then
    openstack flavor create --id 100 --vcpus 1 --ram 256 --disk 1 m1.test
fi

# Generate and add a key pair
# 在启动实例的时候捆绑key到相应的镜像，因为一些镜像没有用户名和密码的登陆方式，必须使用key
if [[ ! $(openstack keypair list | grep testkey) ]]; then
    openstack keypair create --public-key ~/.ssh/id_rsa.pub testkey
fi

# Launch the instance:
if [[ ! $(nova list | grep "ping1") ]]; then
    nova boot --flavor m1.test --image cirros \
        --nic net-id=$(neutron net-list | grep demo-net | awk '{print $2}') \
        --security-group default --key-name testkey ping1
    sleep 10
    # Create a floating IP address and associate it with the instance:
    floating_ip1=$(neutron floatingip-create ext-net \
                | grep floating_ip_address | awk '{print $4}')
    nova floating-ip-associate ping1 $floating_ip1
fi

if [[ ! $(nova list | grep "ping2") ]]; then
    nova boot --flavor m1.test --image cirros \
        --nic net-id=$(neutron net-list | grep demo-net | awk '{print $2}') \
        --security-group default --key-name testkey ping2
    sleep 10
    # Create a floating IP address and associate it with the instance:
    floating_ip2=$(neutron floatingip-create ext-net \
                | grep floating_ip_address | awk '{print $4}')
    nova floating-ip-associate ping2 $floating_ip2
fi

# Ping Test
ssh cirros@$floating_ip1 ping -c 4 $floating_ip2
ssh cirros@$floating_ip2 ping -c 4 $floating_ip1

# Clean the openstack
#openstack keypair delete testkey
#nova delete ping1
#nova delete ping2

set +ex

echo "===== Test Pass! ====="
```

7. 扩展的ansible模块
---------------------
- ansible除了自身内置的module以外，还可以手动用python编写module以扩展功能
- 在compass4nfv的代码里的路径为/compass4nfv/deploy/adapters/ansible/ansible_modules/keystone_endpoint.py
- 这个模块的作用是管理OpenStack的endpoint，操作包括创建，升级和删除
- 可以定义的选项为service_type, enabled, interface, url, region, state
- 这个模块的功能最终是通过OpenStack下面的shade实现的，OpenStack自身有API可供操作
