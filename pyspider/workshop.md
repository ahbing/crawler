[TOC]

# 1. 搭建swarm

## 环境
机器: MAC-10.12.5
docker-machine: 0.10.0, build 76ed2a6
virtualbox: 5.0.28 r111378


## 创建节点
创建三个docker machine:
```
$ docker-machine create --driver virtualbox manager1
$ docker-machine create --driver virtualbox worker1
$ docker-machine create --driver virtualbox worker2
```
执行如下命令可查看新创建的docker machine, 以及docker machine对用的IP地址
```
$ docker-machine ls
NAME       ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
default    *        virtualbox   Running   tcp://192.168.99.100:2376           v17.05.0-ce
manager1   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.05.0-ce
worker1    -        virtualbox   Running   tcp://192.168.99.102:2376           v17.05.0-ce
worker2    -        virtualbox   Running   tcp://192.168.99.103:2376           v17.05.0-ce
```

## 创建swarm
登陆 manager1:
```
$ docker-machine ssh manager1
```
执行如下命令，创建一个新的swarm
```
docker@manager1:~$ docker swarm init --advertise-addr 192.168.99.101

Swarm initialized: current node (wpf2jcvhhvfosv3c9ac6c50dh) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-69wvyxsrnjtm11z38eus20tm0z9cof2ks9khzyv7fdo8it0dln-drdoszuykjp1uvhmn2spaa8vj \
    192.168.99.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

## 将节点加入swarm
登陆worker1, 将worker1加入swarm:

```
docker@worker1:~$ docker swarm join \
>     --token SWMTKN-1-69wvyxsrnjtm11z38eus20tm0z9cof2ks9khzyv7fdo8it0dln-drdoszuykjp1uvhmn2spaa8vj \
>     192.168.99.101:2377
This node joined a swarm as a worker.
```
登陆worker2, 将worker2加入swarm:
```
docker@worker2:~$ docker swarm join \
>     --token SWMTKN-1-69wvyxsrnjtm11z38eus20tm0z9cof2ks9khzyv7fdo8it0dln-drdoszuykjp1uvhmn2spaa8vj \
>     192.168.99.101:2377
This node joined a swarm as a worker.
```

## 查看当前swarm状态
```
docker@manager1:~$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
k926754fhudg5tu51rnlp2fdj     worker2             Ready               Active
q1seyrwugtdceqd515tmp8ph3     worker1             Ready               Active
wpf2jcvhhvfosv3c9ac6c50dh *   manager1            Ready               Active              Leader
```
至此，三个节点的swarm已经创建完成。

# 2. 编写docker-compose.yml

## docker-compose.yml请参考
[docker-compose.yml](https://github.com/pjhu/crawler/blob/master/pyspider/docker-compose.yml)

## 注意事项
- compose file使用version: 3
- 部署多replicas时，不需要使用HAproxy的Load Balance，swarm服务本身自带VIP，如：

![](http://upload-images.jianshu.io/upload_images/5511393-bf2a626bb5a33fed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)
- command中需要连接其它服务时，配置参数不可写在文件中，如scheduler需要连接mysql, redis等，不能使用config.json文件
  ```
  command: -c config.json scheduler
  ```
- 在使用stack deploy时，docker-compose file中 link, depends_on等命令被忽略，具体信息请参考[compose file版本3](https://docs.docker.com/compose/compose-file/)
- 所有服务使用同一网络段，如cars
- 如果需要连接远端mysql, redis修改连接地址即可，如

  ```
  command: '--taskdb "mysql+taskdb://root:root@10.208.20.94:3306/taskdb" --resultdb "mysql+resultdb://root:root@10.208.20.94:3306/resultdb" --projectdb "mysql+projectdb://root:root@10.208.20.94:3306/projectdb" --message-queue "redis://10.208.20.94:6379/db" webui --max-rate 10 --max-burst 3 --scheduler-rpc "http://scheduler:23333/" --fetcher-rpc "http://fetcher/"'
  ```

# 3. 部署服务
## 部署
登陆manager1, 执行如下命令:
```
docker@manager1:~$ docker stack deploy -c docker-compose.yml myspider
```

## 注意事项
swarm服务部署没有严格的顺序，所以会出现mysql, redis服务启动较晚，在service部署要设置restart_policy, 如
```
docker@manager1:~$ docker stack deploy -c docker-compose.yml myspider
Creating network myspider_cars
Creating service myspider_fetcher
Creating service myspider_processor
Creating service myspider_result-worker
Creating service myspider_webui
Creating service myspider_redis
Creating service myspider_mysql
Creating service myspider_scheduler
Creating service myspider_phantomjs
```
