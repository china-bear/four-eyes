## kafka和zookeeper开启kerberos认证

### 1. 环境

kafka版本：2.12-2.3.0

zookeeper版本：3.6.0

操作系统：CentOS7



### 2. 创建主体并生成keytab

在kerberos中，用户和服务是平等的关系，都是以principal的形式存在

```shell
$ kadmin.local
# kafka broker，zookeeper，kafka client主体
# 其中stream.dtwave.local表示kafka broker所在主机的FQDN，Fully Qualified Domain Name的缩写, 含义是完整的域名
$ kadmin.local:  addprinc kafka/stream.dtwave.local@EXAMPLE.COM
$ kadmin.local:  addprinc zookeeper/stream.dtwave.local@EXAMPLE.COM
$ kadmin.local:  addprinc clients/stream.dtwave.local@EXAMPLE.COM

# 生成主体对应的keytab文件
$ kadmin.local:  xst -k /opt/third/kafka/kerberos/kafka_server.keytab
$ kadmin.local:  xst -k /opt/third/zookeeper/kerberos/kafka_zookeeper.keytab
$ kadmin.local:  xst -k /opt/third/kafka/kerberos/kafka_client.keytab

# 给keytab赋予可读权限
$ chmod -R 777 /opt/third/kafka/kerberos/kafka_server.keytab
$ chmod -R 777 /opt/third/zookeeper/kerberos/kafka_zookeeper.keytab
$ chmod -R 777 /opt/third/kafka/kerberos/kafka_client.keytab
```

设置FQDN的方式

```shell
$ cat /etc/hostname
demo-db
$ vim /etc/hosts
192.168.90.88  stream.dtwave.local demo-db
# 192.168.90.88是本机ip，stream.dtwave.local是要设置的FQDN，demo-db是主机名
```



### 3. 配置jaas.conf

/opt/third/kafka/kerberos/kafka_server_jaas.conf

```properties
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/third/kafka/kerberos/kafka_server.keytab"
    principal="kafka/stream.dtwave.local@EXAMPLE.COM";
};
# 此context名字为ZKClient，对应kafka broker启动参数-Dzookeeper.sasl.client=ZkClient
ZkClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/opt/third/kafka/kerberos/kafka_server.keytab"
    principal="kafka/stream.dtwave.local@EXAMPLE.COM";
};
```

/opt/third/zookeeper/kerberos/zookeeper_jaas.conf

```properties
Server {
    com.sun.security.auth.module.Krb5LoginModule required debug=true
    useKeyTab=true
    storeKey=true
    useTicketCache=false
    keyTab="/opt/third/zookeeper/kerberos/kafka_zookeeper.keytab"
    principal="zookeeper/stream.dtwave.local@EXAMPLE.COM";
};
```

/opt/third/kafka/kerberos/kafka_client_jaas.conf

```properties
# 此context名字为KafkaClient，对应kafka consumer启动参数-Dzookeeper.sasl.client=KafkaClient
KafkaClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   storeKey=true
   keyTab="/opt/third/kafka/kerberos/kafka_client.keytab"
   principal="clients/stream.dtwave.local@EXAMPLE.COM";
};
```

### 4. 配置kafka server.properties

```properties
listeners=SASL_PLAINTEXT://stream.dtwave.local:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=GSSAPI
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka
```

### 5. 配置kafka zookeeper.properties

```properties
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
```



### 6. kafka broker+zookeeper启动脚本

```shell
# 启动zookeeper
export KAFKA_OPTS='-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/third/zookeeper/kerberos/zookeeper_jaas.conf'

/opt/third/zookeeper/bin/zkServer.sh start >> /opt/third/kafka/start.log 2>&1

# 启动kafka
export KAFKA_OPTS='-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/third/kafka/kerberos/kafka_server_jaas.conf -Dzookeeper.sasl.client=ZkClient'

JMX_PORT=9988 nohup /opt/third/kafka/bin/kafka-server-start.sh /opt/third/kafka/config/server.properties >> /opt/third/kafka/start.log 2>&1 &
```



### 7. kafka client的使用

通过kafka client与kafka broker进行交互时，之间传输的TGT等信息都需要通过加密算法进行加密，当使用AES256算法加密时，由于受到美国软件出口的管制，需要覆盖`%JAVA_HOME%\jre\lib\security`下的`local_policy.jar`和`US_export_policy.jar`，[下载地址](https://www.7down.com/soft/310593.html)

#### 7.1 producer

```shell
# --bootstrap-server后接FQDN+port，不能接localhost
export KAFKA_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/third/kafka/kerberos/kafka_client_jaas.conf -Dzookeeper.sasl.client=KafkaClient"

sh biu/kafka-console-producer.sh --broker-list localhost:9092 --topic test --producer.config /opt/third/kafka/config/producer.properties
```

其中`producer.properties`的内容如下：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=GSSAPI
sasl.kerberos.service.name=kafka
```



#### 7.2 consumer

```shell
# --bootstrap-server后接FQDN+port，不能接localhost
export KAFKA_OPTS="-Djava.security.krb5.conf=/etc/krb5.conf -Djava.security.auth.login.config=/opt/third/kafka/kerberos/kafka_client_jaas.conf -Dzookeeper.sasl.client=KafkaClient"

sh bin/kafka-console-consumer.sh --bootstrap-server stream.dtwave.local:9092 --topic test --from-beginning --consumer.config /opt/third/kafka/config/consumer.properties
```

其中，`consumer.properties`的内容如下：

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=GSSAPI
sasl.kerberos.service.name=kafka
```

> 参考：
>
> https://simple.wikipedia.org/wiki/Kerberos_(protocol)
>
> https://www.orchome.com/436
>
> https://community.cloudera.com/t5/Support-Questions/Kafka-client-code-does-not-currently-support-obtaining-a/td-p/283879
>
> https://www.cnblogs.com/bdicaprio/articles/10096250.html
>
> https://stackoverflow.com/questions/43469962/kafka-sasl-zookeeper-authentication
>
> https://docs.confluent.io/4.1.1/kafka/authentication_sasl_gssapi.html#
>
> https://www.orchome.com/326
>
> https://www.orchome.com/1944
>
> https://www.orchome.com/500