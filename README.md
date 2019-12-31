# 一、Kafka Monitoring

### 1. First you have to install zabbix-java-gataway（zabbix-server服务端）
    yum install -y zabbix-java-gataway
### 2. Configuring zabbix-java-gataway
    vim /etc/zabbix/zabbix_java_gateway.conf
Uncoment and set **START_POLLERS=10**
### 3. Configuring zabbix-server
    vim /etc/zabbix/zabbix_server.conf
Uncoment and set to **StartJavaPollers=5**
Change IP for **JavaGateway=IP_address_java_gateway**
### 4. Restart zabbix-server
    systemctl restart zabbix-server
### 5. Add to autorun zabbix-java-gataway
     chkconfig zabbix-java-gataway on
### 6. Start zabbix-java-gataway
    systemctl start zabbix-java-gateway
### 7. Kafka configuration（kafka服务器，zabbix-agent端）

##### 7.1 修改配置文件 `/data/kafka/kafka/bin/kafka-run-class.sh`

    vim /data/kafka/kafka/bin/kafka-run-class.sh;

change from

    # JMX settings
    if [ -z "$KAFKA_JMX_OPTS" ]; then
       KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false "
    fi

to

    # JMX settings
    if [ -z "$KAFKA_JMX_OPTS" ]; then
       KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=12345 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false "
    fi

##### 7.2 修改配置文件 `/data/kafka/kafka/bin/kafka-server-start.sh`

    vim /data/kafka/kafka/bin/kafka-server-start.sh;

change from

    if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
        export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
    fi

to

    if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
        export KAFKA_HEAP_OPTS="-Xmx1G -Xms512M"
    fi
    
### 8. 安装守护进程supervisor服务,并将kafka添加到守护进程中

##### 8.1 安装supervisor
    yum install supervisor -y  

##### 8.2 增加kafka启动配置参数

```
cd /etc/supervisord.d;
vim /etc/supervisord.d/kafka.ini;
```

    [program:kafka]
    command=/data/kafka/kafka/bin/kafka-server-start.sh /data/kafka/kafka/config/server.properties
    directory=/data/kafka/kafka/
    autostart=true
    autorestart=true
    stopasgroup=true
    startsecs=10
    startretries=999
    user=kafka
    log_stdout=true
    log_stderr=true
    logfile=/data/kafka/kafka/log/supervisord-kafka.out
    logfile_maxbytes=20MB
    logfile_backups=10
 
##### 8.3 重启supervisor 
    systemctl enable supervisord.service;
    systemctl start supervisord.service;
    systemctl restart supervisord.service;
    systemctl status supervisord.service;

### 9. 设置添加Kafka为服务

本例子中，kafka安装在目录：`/data/kafka/kafka`

##### 9.1 新增kafka服务配置文件
```
cd /lib/systemd/system;
vim /lib/systemd/system/kafka.service;
```
添加以下内容（需要根据实际参数进行更改）
```
[Unit]
Description=Apache Kafka
Wants=network.target
After=network.target

[Service]
LimitNOFILE=32768
User=kafka

Environment=JAVA=/usr/bin/java
Environment="KAFKA_USER=kafka"
Environment="KAFKA_HOME=/data/kafka/kafka"
Environment="SCALA_VERSION=2.11"
Environment="KAFKA_CONFIG=/data/kafka/kafka/config"
Environment="KAFKA_BIN=/data/kafka/kafka/bin"
Environment="KAFKA_LOG4J_OPTS=-Dlog4j.configuration=file:/data/kafka/kafka/config/log4j.properties"
Environment="KAFKA_OPTS="
Environment="KAFKA_HEAP_OPTS=-Xmx1G -Xms512M"
Environment="KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseCompressedOops -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+CMSScavengeBeforeRemark -XX:+DisableExplicitGC -Djava.awt.headless=true"
Environment="KAFKA_LOG_DIR=/data/kafka/kafka/log"
Environment="KAFKA_JMX_OPTS=-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.net.preferIPv4Stack=true -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.port=12345 -Dcom.sun.management.jmxremote.rmi.port=12345 -Djava.rmi.server.hostname=10.20.22.12"

ExecStart=/data/kafka/kafka/bin/kafka-server-start.sh ${KAFKA_CONFIG}/server.properties

SuccessExitStatus=0 143

Restart=on-failure
RestartSec=15

[Install]
WantedBy=multi-user.target
```
特别注意以下参数：
```
 -Dcom.sun.management.jmxremote.port=
 -Dcom.sun.management.jmxremote.rmi.port=
 -Djava.rmi.server.hostname=
```
To avoid using dynamic TCP port and firewall/NAT issues it is better to set static port setting.

###### 9.2 重启kafka服务
    systemctl daemon-reload;
    systemctl restart kafka;

注：若`/lib/systemd/system/kafka.service`中的内容有更改，则需要执行`systemctl daemon-reload`

### 10. zabbix监控配置

#Upload scripts for discovery JMX

    mkdir -p /usr/lib/zabbix/externalscripts/kafka;cd /usr/lib/zabbix/externalscripts/kafka;
    git clone https://github.com/zhanglh0307/kafka-monitoring.git;
    cp -r zabbix/kafka/jmx_discovery /usr/lib/zabbix/externalscripts/kafka;
    cp -r zabbix/kafka/JMXDiscovery-0.0.1.jar /usr/lib/zabbix/externalscripts/kafka;

##Import template（上传模板）
Log in to your zabbix web

**Click Configuration->Templates->Import**

Download template [zbx_kafka_templates.xml](https://github.com/zhanglh0307/kafka-monitoring/blob/master/zbx_kafka_templates.xml) and upload to zabbix
Then add this template to Kafka and configure JMX interfaces on zabbix 

Enter Kafka IP address and JMX port
If you see jmx icon, you configured JMX monitoring  good!

### 11. Troubles 
if you have problems you can check JMX using this script
     
     #!/usr/bin/env bash
     
     ZBXGET="/usr/bin/zabbix_get"
     if [ $# != 5 ]
     then
     echo "Usage: $0 <JAVA_GATEWAY_HOST> <JAVA_GATEWAY_PORT> <JMX_SERVER> <JMX_PORT> <KEY>"
     exit;
     fi
     QUERY="{\"request\": \"java gateway jmx\",\"conn\": \"$3\",\"port\": $4,\"keys\": [\"$5\"]}"
     $ZBXGET -s $1 -p $2 -k "$QUERY"

**eg.:**
    
    ./zabb_get_java  zabbix-java-gateway-ip 10052 server-test-ip 12345 'jmx[java.lang:type=Threading,PeakThreadCount]'

For monitoring kafka consumers you should install [Burrow](https://github.com/linkedin/Burrow/) daemon and [jq](https://stedolan.github.io/jq/download/) tools on kafka host

***

# 二、Kafka Consumer Monitoring

### Clone all stuff 
    mkdir -p /usr/lib/zabbix/externalscripts/kafka;cd /usr/lib/zabbix/externalscript/kafka;    
    ssh clone https://github.com/zhanglh0307/kafka-monitoring.git
    cd kafka/kafkaconsumers
     
### Install burrow
    cp -r burrow /opt/
    cp burrow/ /etc/init.d/burrow_script
    chkconfig --level 345 burrow_script on
You should change config file in /opt/burrow/burrow.cfg

### Install jq 
    cd /usr/bin
    wget https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64
    mv jq-linux64 jq
    chmod +x jq
     
### Check jq

    # jq 
     jq - commandline JSON processor [version 1.5]
     Usage: jq [options] <jq filter> [file...]
     jq is a tool for processing JSON inputs, applying the
     given filter to its JSON text inputs and producing the
     filter's results as JSON on standard output.
     The simplest filter is ., which is the identity filter,
     copying jq's input to its output unmodified (except for
     formatting).
     For more advanced filters see the jq(1) manpage ("man jq")
     and/or https://stedolan.github.io/jq
     Some of the options include:
     -c compact instead of pretty-printed output;
     -n use `null` as the single input value;
     -e set the exit status code based on the output;
     -s read (slurp) all inputs into an array; apply filter to it;
     -r output raw strings, not JSON texts;
     -R read raw strings, not JSON texts;
     -C colorize JSON;
     -M monochrome (don't colorize JSON);
     -S sort keys of objects on output;
     --tab use tabs for indentation;
     --arg a v set variable $a to value <v>;
     --argjson a v set variable $a to JSON value <v>;
     --slurpfile a f set variable $a to an array of JSON texts read from <f>;
     See the manpage for more options.
     
### Copy files to zabbix folder
     cp kafka_consumers.sh /etc/zabbix/
     cp userparameter_kafkaconsumer.conf /etc/zabbix/zabbix_agentd.d
     Start burrow and restart zabbix-agent
     /etc/init.d/burrow_script start
     /etc/init.d/zabbix-agent restart
Upload template [zbx_templates_kafkaconsumers.xml](https://github.com/zhanglh0307/kafka-monitoring/blob/master/kafkaconsumers/zbx_templates_kafkaconsumers.xml) and mapping value [zbx_valuemaps_kafkaconsumers.xml](https://github.com/zhanglh0307/kafka-monitoring/blob/master/kafkaconsumers/zbx_valuemaps_kafkaconsumers.xml) to zabbix server using UI and link template to Kafka host
### Troubleshooting 
If it doesn't work you can check it use `/etc/zabbix/kafka_consumers.sh`  
e.g.:

   ` # /etc/zabbix/kafka_consumers.sh discovery`
    
    {"data":[{
    "{#CONSUMER}": "CONSUMER0",
    "{#PARTITION}": 0,
    "{#TOPIC}": "TOPIC0"
    },{
    "{#CONSUMER}": "CONSUMER1",
    "{#PARTITION}": 0,
    "{#TOPIC}": "TOPIC1"
    }]}


## kafka-monitoring相关
http://adminotes.com/

https://github.com/zhanglh0307/kafka-monitoring/wiki/Kafka-monitoring

https://engineering.linkedin.com/apache-kafka/burrow-kafka-consumer-monitoring-reinvented

https://github.com/linkedin/Burrow/wiki

https://community.hortonworks.com/articles/28103/monitoring-kafka-with-burrow.html

https://github.com/RiotGamesMinions/zabbix_jmxdiscovery
