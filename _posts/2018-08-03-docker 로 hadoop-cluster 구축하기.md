---
tags:
  - hadoop
  - docker
  - network
  - subnetmask
---



## 개요

현재 제가 사용 가능한 서버는 하나입니다. 

여기에 docker 를 이용해, master 1개에 slave 20 개를 가진 hadoop cluster 를 만드는 작업을 해 보았습니다.



## 진행

### hadoop image

 이미지는 저희 랩실에서 만들어둔 이미지를 사용하였습니다. 사용 이미지는

https://hub.docker.com/r/kmubigdata/ubuntu-hadoop/

에서 받으실 수 있습니다.

Dockerfile 은 다음과 같습니다.

```dockerfile
FROM kmubigdata/ubuntu-1604
MAINTAINER kimjeongchul

USER root

# install dev tools
RUN apt-get install -y curl openssh-server openssh-client rsync wget

# passwordless ssh
RUN rm -f /etc/ssh/ssh_host_dsa_key /etc/ssh/ssh_host_rsa_key /root/.ssh/id_rsa
RUN ssh-keygen -q -N "" -t dsa -f /etc/ssh/ssh_host_dsa_key
RUN ssh-keygen -q -N "" -t rsa -f /etc/ssh/ssh_host_rsa_key
RUN ssh-keygen -q -N "" -t rsa -f /root/.ssh/id_rsa
RUN cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys

# java
RUN mkdir -p /usr/java/default
RUN wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz
RUN tar -xzvf jdk-8u131-linux-x64.tar.gz -C /usr/java/default --strip-components=1

# java env
ENV JAVA_HOME /usr/java/default
ENV PATH $PATH:$JAVA_HOME/bin

RUN rm -rf /usr/bin/java
RUN ln -s $JAVA_HOME/bin/java /usr/bin/java

# hadoop
RUN wget http://apache.mirror.cdnetworks.com/hadoop/common/hadoop-2.8.0/hadoop-2.8.0.tar.gz
RUN tar -xvzf hadoop-2.8.0.tar.gz -C /usr/local/
RUN cd /usr/local && ln -s ./hadoop-2.8.0 hadoop
RUN rm hadoop-2.8.0.tar.gz

# hadoop env
ENV HADOOP_HOME /usr/local/hadoop
ENV HADOOP_PREFIX $HADOOP_HOME
ENV HADOOP_COMMON_HOME $HADOOP_HOME
ENV HADOOP_HDFS_HOME $HADOOP_HOME
ENV HADOOP_MAPRED_HOME $HADOOP_HOME
ENV HADOOP_YARN_HOME $HADOOP_HOME
ENV HADOOP_CONF_DIR $HADOOP_HOME/etc/hadoop
ENV YARN_CONF_DIR $HADOOP_HOME/etc/hadoop
ENV PATH $PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

RUN sed -i '/^export JAVA_HOME/ s:.*:export JAVA_HOME=/usr/java/default\nexport HADOOP_PREFIX=/usr/local/hadoop\nexport HADOOP_HOME=/usr/local/hadoop\n:' $HADOOP_PREFIX/etc/hadoop/hadoop-env.sh
RUN sed -i '/^export HADOOP_CONF_DIR/ s:.*:export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop/:' $HADOOP_PREFIX/etc/hadoop/hadoop-env.sh

RUN mkdir -pv $HADOOP_HOME/input
RUN mkdir -pv $HADOOP_HOME/dfs
RUN mkdir -pv $HADOOP_HOME/dfs/name
RUN mkdir -pv $HADOOP_HOME/dfs/data
RUN mkdir -pv $HADOOP_HOME/tmp

RUN cp $HADOOP_HOME/etc/hadoop/*.xml $HADOOP_HOME/input

# pseudo distributed
ADD hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml
ADD core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml
ADD mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml
ADD yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml
ADD slaves $HADOOP_HOME/etc/hadoop/slaves

RUN $HADOOP_HOME/bin/hdfs namenode -format

ADD ssh_config /root/.ssh/config
RUN chmod 600 /root/.ssh/config
RUN chown root:root /root/.ssh/config

# workingaround docker.io build error
RUN ls -la $HADOOP_HOME/etc/hadoop/*-env.sh
RUN chmod +x $HADOOP_HOME/etc/hadoop/*-env.sh
RUN ls -la $HADOOP_HOME/etc/hadoop/*-env.sh

# fix the 254 error code
RUN sed  -i "/^[^#]*UsePAM/ s/.*/#&/"  /etc/ssh/sshd_config
RUN echo "UsePAM no" >> /etc/ssh/sshd_config
RUN echo "Port 2122" >> /etc/ssh/sshd_config

COPY bootstrap.sh /etc/bootstrap.sh
RUN chown root.root /etc/bootstrap.sh
RUN chmod 700 /etc/bootstrap.sh

# HDFS ports
EXPOSE 50010 50020 50070 50075 50090 8020 9000

# Mapred ports
EXPOSE 10020 19888

# YARN ports
EXPOSE 8030 8031 8032 8033 8040 8042 8088

# Other ports
EXPOSE 49707 2122

ENTRYPOINT ["/etc/bootstrap.sh"]
```



### docker network 설정

hadoop clustering 을 위해서는 컨테이너들을 같은 네트워크에 묶어줘야 합니다.

`docker network create --subnet 10.0.2.0/24 hadoop-cluster`



#### 서브넷 마스크란

기본적으로 서브넷은 너무 큰 네트워크를 효율적으로 나눠 쓰기 위해 도입된 개념입니다.

ip는 2진수 32자리로 구성되어 있는데, 이걸 그냥 통째로 나누면 2^32 ,42억개라는 값이 나옵니다.

그런데 한 사람이 필요도 없이 너무 많이 할당받는 경우가 생겨서, 클래스 개념이 도입되었습니다.

Class 는 A,B,C의 3개로 나뉘어져 있고 C로 갈수록 할당받을 수 있는 host 갯수가 작아집니다.

![](https://www.dropbox.com/s/3wxaehdmk5do7vm/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-03%2015.46.32.jpg?raw=1)

A는 2^24(8+8+8)개의 ip를 할당 받을 수 있는 반면, C는 2^8 만 할당 가능합니다.

그런데 C를 쓰더라도 256개나 되서, 더 조금만 필요한 경우가 많이 나왔습니다.

그래서 subnet mask 를 도입하게 됩니다.



![](https://www.dropbox.com/s/ngkjr468crxls0z/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-03%2015.48.24.jpg?raw=1)

당연히 class보다 더 잘게 쪼개야 합니다.

그림을 보시면 host를 또 쪼개서, subnet number가 추가된 것을 보실 수 있습니다.



![](https://www.dropbox.com/s/omxcl7ady28l16n/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-03%2015.50.48.jpg?raw=1)

class에서 host 부분을 우리의 필요에 따라 잘게 쪼개는 모습입니다.

2진수기 때문에, 2의 n승 갯수로만 쪼갤 수 있습니다.

1개부터 255개까지 내가 필요한 갯수보다 한단계 더 큰 갯수가 나오도록 쪼개면 되겠죠!

그리고 규칙성이 하나더 발견되는데, 00000000 에서 왼쪽부터 1이 채워진다는 것입니다.

이 규칙을 이용해서, subnet mask를 다 표기하는 대신 1의 갯수만 ip주소 옆에 쓰기도 합니다.

10.0.2.0/24

위에서 docker network 설정할 때 쓴 주소입니다.

해석해 보면, 10.0.2.0 의 subnet mask는 1이 24개, 즉 11111111 11111111 11111111 00000000 = 255.255.255.0

즉 0이 8개이므로 2^8 = 256 개의 host 를 쓸 수 있다는 뜻입니다.



### docker run

마스터 노드 1개와 slave 노드 20개를 수동으로 해 주었습니다.

#### master

```
docker run -dit --name yarn-master --network hadoop-cluster -p 12345:8088 --ip 10.0.2.2 --add-host=master:10.0.2.2 --add-host=slave1:10.0.2.3 --add-host=slave2:10.0.2.4 --add-host=slave3:10.0.2.5 --add-host=slave4:10.0.2.6 --add-host=slave5:10.0.2.7 --add-host=slave6:10.0.2.8 --add-host=slave7:10.0.2.9 --add-host=slave8:10.0.2.10 --add-host=slave9:10.0.2.11 --add-host=slave10:10.0.2.12 --add-host=slave11:10.0.2.13 --add-host=slave12:10.0.2.14 --add-host=slave13:10.0.2.15 --add-host=slave14:10.0.2.16 --add-host=slave15:10.0.2.17 --add-host=slave16:10.0.2.18 --add-host=slave17:10.0.2.19 --add-host=slave18:10.0.2.20 --add-host=slave19:10.0.2.21 --add-host=slave20:10.0.2.22 kmubigdata/ubuntu-hadoop /bin/bash
```

위에서 만든 네트워크 안에 같은 subnet으로 컨테이너들을 --add-host 명령어를 반복해 묶어 주는 모습입니다.

그리고 run 시점에서 다 고정 ip를 사용해 주므로, --add-host 는 /etc/hosts 에 해당 ip를 추가해 주는 명령어입니다. /etc/hosts 파일은 이름과 ip 주소를 매칭시켜주는 역할을 하는 파일입니다.

유의하실 점은 기본 포트인 8088은 유명해서 해커들의 표적이 될 수 있기 때문에, -p 옵션을 이용해 포트포워딩을 해 줘야 합니다.



#### slave

```
docker run -dit --name yarn-slave1 --network hadoop-cluster --ip 10.0.2.3 --add-host=master:10.0.2.2 --add-host=slave1:10.0.2.3 --add-host=slave2:10.0.2.4 --add-host=slave3:10.0.2.5 --add-host=slave4:10.0.2.6 --add-host=slave5:10.0.2.7 --add-host=slave6:10.0.2.8 --add-host=slave7:10.0.2.9 --add-host=slave8:10.0.2.10 --add-host=slave9:10.0.2.11 --add-host=slave10:10.0.2.12 --add-host=slave11:10.0.2.13 --add-host=slave12:10.0.2.14 --add-host=slave13:10.0.2.15 --add-host=slave14:10.0.2.16 --add-host=slave15:10.0.2.17 --add-host=slave16:10.0.2.18 --add-host=slave17:10.0.2.19 --add-host=slave18:10.0.2.20 --add-host=slave19:10.0.2.21 --add-host=slave20:10.0.2.22 kmubigdata/ubuntu-hadoop /bin/bash
```

 master와 마찬가지 입니다.



![](https://www.dropbox.com/s/z0nlniab3lngglo/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-02%2023.44.44.jpg?raw=1)

docker ps 한 모습입니다. 



### start-all.sh

master 노드에서 전체를 실행합시다.

`cd $HADOOP_PREFIX/sbin `

`./start-all.sh`



![](https://www.dropbox.com/s/nfje0gzd76rfprs/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202018-08-03%2001.43.05.jpg?raw=1)

master의 8088 포트를 열어놓은 이유는 , 웹으로 하둡 클러스터 상태를 확인하기 위해서입니다.

해당 서버 주소:8088 로 접속하시면, 현재 노드들의 상태를 확인하실 수 있습니다.





## 결론

힘든 작업이지만 연구실 선배님들의 도움을 받아 해볼 수 있었습니다.

docker network를 swarm 으로 구성하지 않아, 다른 서버와의 오케스트레이션은 할 수 없습니다.

나중에 기회가 되면 swarm 으로 구성해볼 예정입니다.

다음 포스팅에서는 위에서 이용한 hadoop 이미지에 mongo-hadoop connector 를 설치하고 데이터를 hdfs를 거치지 않고 불러오는 작업을 해 보겠습니다.

