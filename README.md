# blog
需要考虑的问题：  
1 日志,快照等如何存放；  
2 同一个镜像怎么做到个性化配置；  
3 外部如何访问zk集群？
首先完成镜像制作，单机版的Dockfile如下:
FROM centos  
MAINTAINER 44917134@qq.com  
USER root

\# 更新源

RUN rm /etc/yum.repos.d/CentOS-Base.repo -rf

ADD CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo

RUN yum -y install --enablerepo base wget java tar.x86_64 && wget -q -O - http://172.28.0.2:8080/zookeeper-3.4.6.tar.gz | tar -xzf - -C /opt && mv /opt/zookeeper-3.4.6 /opt/zookeeper && cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg && mkdir -p /opt/zookeeper/data && mkdir -p /opt/zookeeper/log

ENV JAVA_HOME /usr/

ADD start.sh /start.sh

WORKDIR /opt/zookeeper

\# 用来记录日志和快照及传递公用配置的卷

VOLUME ["/opt/zookeeper/conf", "/opt/zookeeper/data", "/opt/zookeeper/log"]

ENTRYPOINT ["/start.sh"]

\# 保证前台运行

CMD ["start-foreground"]

执行完成镜像制作:

docker build -f Dockerfile -t zookeeper:v2 .

启动容器：
    docker  run -i -t -d -p 172.28.2.26:2181:2181  zookeeper:v2   

访问zk：
     ./zkCli.sh  -server  172.28.2.26:2181    

上面只是完成了单机模式的zk，对于集群模式的zk使用docker来做的话就需要考虑同一个镜像每个容器的个性化配置。对于zk集群需要考虑的是：  
1）集群中每个节点的myid  
2）集群中其他server的ip端口信息  
考虑使用环境变量的方式来导入集群中每个server的个性化配置。但是，zk的配置文件还存在一些公用的配置，对于这种共有的配置还通过环境变量导入的话，会显得命令格外的冗长。有种方式是直接固化到容器镜像当中，但是如果后续希望调整共有配置的话又必须重新制作容器不够灵活。因此，考虑这种共有配置通过挂载一个host上的文件传入。 当然还有一种选择是通过在容器内部署一个版本管理工具如git或者配置中心zk,etcd等，这个配置管理的agent可以部署的容器内，这样就不需要挂载共有配置数据卷或者用配置管理软件管理host上的共有配置，这样不需要在容器里面多运行一个代理。
