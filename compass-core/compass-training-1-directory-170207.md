Compass培训第一节——目录详解
==================================================

1. 简介与范围
----------------------
- 这节主要涉及compass下的bin和conf的目录，对其中各文件及其作用进行解读
- 代码源：https://github.com/openstack/compass-core/tree/dev/remote-deploy
- 注意讲解的代码为`dev/remote-deploy`的branch，而非master


2. `/bin`目录及文件
------------------------
```
|--compass-core
   |--ansible_callbacks
      |--playbook_done.py
        *当ansible的playbook运行结束之后需要调用的程序，可以返回安装成功或失败的状态*
   |--chef
        *已过期*
   |--cobbler
      |--remove_systems.sh
        *清空在cobbler中已经注册过的hosts*
   |--clean_installation_logs.py
      *清空所有安装日志*
   |--clean_installers.py
      *清空installer中已经加载的配置内容，installer包括os和package的installer*
   |--client.py
      *应放置在/compass/apiclient中，是一个通过python的library调用api来进行部署的流程*
   |--client.sh
      *通过命令行向client.py传入配置参数完成部署*
   |--compass_check.py
      *检验compass是否已经被正确和完整安装*
   |--compass_wsgi.py
      *compass的webapp的部分，compass使用Python的Flask模块提供服务*
   |--compassd  
      *已过期*
   |--csvdeploy.py
      *通过将配置写入csv文件进行部署*
   |--delete_clusters.py
      *将特定的cluster的信息从数据库中删除，不影响其他如adapter的信息*
   |--manage_db.py
      *数据库的创建和重构*
   |--poll_switch.py
      *因为涉及到不同的交换机和通信标准，这部分不再支持*
   |--progress_update.py
      *安装进度更新，将安装进度计算的任务交给celery*
   |--query_switch.py
      *用于测试poll switch的功能，已过期*
   |--refresh.sh
      *调用refresh_agent.sh和refresh_server对compass进行重置，恢复初始状态*
   |--refresh_agent.sh
      *重置ansible，重新启动agent的所有相关service*
   |--refresh_server.sh
      *重置数据库，重新启动server的所有相关service*
   |--runserver.py
      *运行compass server*
   |--switch_virtualenv.py.template
      *切换至python的虚拟环境*
```

3. `/conf`目录及文件
--------------------------
```
|--compass-core
   |--conf
      |--adapter
        *解释和配置了在部署之后需要达到的状态，包括openstack的版本，os和package的installer，系统版本，同时也包含一些基类，例如ceph*
      |--distributed_system
          *如果涉及到其他类分布式系统的安装，例如Hadoop等，需要用到的基类*
      |--flavor
          *在adapter配置的基础上又详细了一层，涉及到在cluster中可以部署的host对应的角色*
      |--flavor_field
          *已过期*
      |--flavor_mapping
          *将从Web UI中读入的flavor的数据写入后端的配置文件*
      |--flavor_metadata
          *规定了flavor配置数据必须提供的配置项*
      |--machine_list
          *如果需要将机器批量写入，通过这个文件*
      |--os
          *规定了能提供部署的所有可能的操作系统*
      |--os_field
          *在global os config涉及到的配置项*
      |--os_installer
          *这里使用了cobbler作为os installer，规定了cobbler的位置和登录用户名及密码*
      |--os_mapping
          *同flavor_mapping的作用类似，这里deploy type的配置项不再使用*
      |--os_metadata
          *verify os配置的数据*
      |--package_field
          *规定了package配置需要的配置项*
      |--package_installer
          *package installer使用ansible，针对不同的openstack版本，规定了ansible及其相关文件的路径*
      |--package_metadata
          *verify package配置的数据*
      |--progress_calculator
          *progress update功能的配置文件，针对的是chef*
      |--role
          *对于每个openstack版本可以用到的所有角色*
      |--switch_list
          *作用类似于machine_list*
      |--templates
          *Python cheetah的模板，对ansible和cobbler这两个installer安装需要的文件模板*
         |--ansible_installer
            |--openstack_juno
               |--ansible_cfg
                  *ansible的配置文件模板*
               |--hosts
                  *规定可使用的机器*
               |--inventories
                  *规定ansible运行需要操作的机器*
               |--vars
                  *ansible运行中需要的参数*
         |--cobbler
      |--celeryconfig
          *celery server的配置，包括url，可接受文件格式等*
      |--setting
          *基础配置，包括数据库，ORM，celery和compass的一些配置*
```
