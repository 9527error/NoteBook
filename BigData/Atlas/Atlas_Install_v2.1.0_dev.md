
# -------------------------------------------------------------------
# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 作者：luoyoumou 钉钉群：30931114 ( 万峰大数据技术交流 )

# 参考：https://docs.cloudera.com/documentation/enterprise/latest/topics/cm_mc_solr_service.html
#      https://atlas.apache.org/2.0.0/HighAvailability.html
#      https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/zookeeper-acls/content/zookeeper_acls_best_practices_atlas.html
#      https://www.programmersought.com/article/19537357119/
#      https://atlas.apache.org/2.0.0/Hook-Hive.html
#      https://atlas.apache.org/2.0.0/Bridge-Kafka.html
#      https://atlas.apache.org/2.0.0/Configuration.html
#      https://jiezhi.github.io/2020/03/31/atlas2-cdh6-hive-hook
#      https://github.com/apache/atlas
#      https://cloud.tencent.com/developer/article/1678920
#      https://blog.csdn.net/fairynini/article/details/106134361
#      https://atlas.apache.org/1.1.0/Security.html
#      https://docs.cloudera.com/runtime/7.2.9/atlas-securing/topics/atlas-configure-kerberos-authentication.html
#      https://blog.csdn.net/qq_36389344/article/details/107233992
#      https://www.cnblogs.com/ttzzyy/p/14143508.html

# 备注1: 大数据 CDH 6.3.2 Kerberos 环境
# 备注2：在 onedts-dev-cdh-cm-v01 服务器编译, 在 onedts-dev-cdh-client-v01, onedts-dev-cdh-client-v02 两台服务器安装（高可用)
# 备注3: ATLAS_HOME 目录是 /DATA/disk1/apps/atlas （ 即 /DATA/disk1/apps/apache-atlas-2.0.0 ) 该目录只需在 "HiveSerer2 服务所在的服务器" 和 "Atlas安装的服务器"上有即可。但 /DATA/disk1/apps/atlas/conf/atlas-application.properties 配置文件在这两者上有所不同

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 1. build atlas
## 1.1 download atlas
```
atlas_base_dir=/DATA/disk1/software/atlas;
atlas_conf_dir=${atlas_base_dir}/atlas_conf_210_bak;
mkdir -p ${atlas_conf_dir}/patch;

cd ${atlas_base_dir};
wget https://mirrors.bfsu.edu.cn/apache/atlas/2.1.0/apache-atlas-2.1.0-sources.tar.gz

tar -xvf apache-atlas-2.1.0-sources.tar.gz;

cd ${atlas_conf_dir};
\cp ../apache-atlas-sources-2.1.0/pom.xml ./;
\cp ../apache-atlas-sources-2.1.0/addons/hive-bridge/src/main/java/org/apache/atlas/hive/bridge/HiveMetaStoreBridge.java ./;

```

# -------------------------------------------------------------------
## 1.2 vi ${atlas_conf_dir}/pom.xml # Due to integration with CDH 6.3.2, inrepositoriesThe following sections are added:
## 在  <repositories></repositories> 中添加如下内容
```
    <repositories>
        <repository>
            <id>cloudera</id>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
```

# -------------------------------------------------------------------
## 1.3 vi ${atlas_conf_dir}/pom.xml # Due to integration with CDH 6.3.2, inrepositoriesThe following sections are added: 
## （ 注意：Hive 保持原来版本 )
## 在  <repositories></repositories> 中添加如下内容
```
        <lucene-solr.version>7.4.0-cdh6.3.2</lucene-solr.version> # 原来 7.3.0 # 7.4.0-cdh6.3.2
        <hadoop.version>3.0.0-cdh6.3.2</hadoop.version>           # 原来 3.1.1   # 3.0.0-cdh6.3.2
        <hbase.version>2.1.0-cdh6.3.2</hbase.version>             # 原来 2.0.2   # 2.1.0-cdh6.3.2
        <solr.version>7.4.0-cdh6.3.2</solr.version>               # 原来 7.5.0   # 7.4.0-cdh6.3.2
        <hive.version>3.1.0</hive.version>                        # 原来 3.1.0   # 不要修改 
        <kafka.version>2.1.0</kafka.version>                      # 原来 2.1.0   # 2.2.1-cdh6.3.2 # 不要修改，否则：Atlas 高可用配置有问题!!!
        <kafka.scala.binary.version>2.11</kafka.scala.binary.version>
        <calcite.version>1.16.0</calcite.version>
        <zookeeper.version>3.4.5-cdh6.3.2</zookeeper.version>     # 原来 3.4.6  # 3.4.5-cdh6.3.2
        <falcon.version>0.8</falcon.version>
        <sqoop.version>1.4.7-cdh6.3.2</sqoop.version>             # 原来 1.4.6.2.3.99.0-195 # 1.4.7-cdh6.3.2
        <storm.version>1.2.0</storm.version>
        <curator.version>4.0.1</curator.version>
        <elasticsearch.version>5.6.4</elasticsearch.version>      # 原来 5.6.4  # 先不修改

```

# -------------------------------------------------------------------
## 1.4 vi +577 ${atlas_conf_dir}/HiveMetaStoreBridge.java # 忽略：该版本无此函数
##     修改 getDatabaseName() 函数，如下
```
    public static String getDatabaseName(Database hiveDB) {
        String dbName      = hiveDB.getName().toLowerCase();
        //String catalogName = hiveDB.getCatalogName() != null ? hiveDB.getCatalogName().toLowerCase() : null;
        String catalogName = null;
        try {
            if (hiveDB.getCatalogName() != null) {
                catalogName = hiveDB.getCatalogName().toLowerCase();
            }
        } catch (NoSuchMethodError e) {
            LOG.warn("Failed while getting catalog name of database");
        }
        if (StringUtils.isNotEmpty(catalogName) && !StringUtils.equals(catalogName, DEFAULT_METASTORE_CATALOG)) {
            dbName = catalogName + SEP + dbName;
        }
        return dbName;
    }

```

# -------------------------------------------------------------------
## 1.8 编译 Atlas # 编译好的位置在 distro/target/ 目录下
## 注意: ATLAS-1546 bug 已经在 2.1.0 版本修复，故：无需考虑该 bug
```

atlas_base_dir=/DATA/disk1/software/atlas;
atlas_conf_dir=${atlas_base_dir}/atlas_conf_210_bak;

cd ${atlas_base_dir};
rm -rf apache-atlas-sources-2.1.0;
tar -xvf apache-atlas-2.1.0-sources.tar.gz;

cd ${atlas_base_dir}/apache-atlas-sources-2.1.0/;
\cp ${atlas_conf_dir}/pom.xml ./pom.xml;
\cp ${atlas_conf_dir}/HiveMetaStoreBridge.java ./addons/hive-bridge/src/main/java/org/apache/atlas/hive/bridge/;

cd ${atlas_base_dir}/apache-atlas-sources-2.1.0/;
export MAVEN_OPTS="-Xms2g -Xmx2g"
# 注意下面得分两步走：先 install, 再 package
mvn clean -DskipTests install
mvn clean -DskipTests package -Pdist

# 下面是一些测试命令，请忽略
# mvn clean dependency:tree -Dincludes='*jackson*' -DskipTests install 
# mvn clean dependency:tree -Dincludes='*jackson*' -DskipTests package -Pdist
# mvn clean -DskipTests package -Pdist -X -T 8
# 调试模式带 -X 参数
# mvn clean -DskipTests package -Pdist -X

# 注意：编译成功后生成的文件在 ./distro/target 目录

```

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 2. Install atlas

# -------------------------------------------------------------------
## 2.1 Install Solr 略

# -------------------------------------------------------------------
## 2.2 Install atlas

### 2.2.1 Install atlas
### 备注：将编译好的 apache-atlas-2.1.0-bin.tar.gz 解压到 /DATA/disk1/apps/ 目录下，然后软链为 atlas，类似如下
###      即 ATLAS_HOME=/DATA/disk1/apps/atlas
```
[root@onedts-dev-cdh-client-v01 ~]# ls -l /DATA/disk1/apps/|grep atlas
drwxr-xr-x 11 atlas      atlas           4096 6月  28 09:01 apache-atlas-2.0.0
lrwxrwxrwx  1 root       root              20 7月  12 17:23 atlas -> ./apache-atlas-2.1.0
```

### 2.2.2 vi $ATLAS_HOME/conf/atlas-env.sh 
### 注意下面文件中涉及到一些目录需要创建，
### 如: mkdir -p /var/log/atlas;chown atlas:hadoop /var/log/atlas;
###     mkdir -p /var/run/atlas;chown atlas:hadoop /var/run/atlas;
```

export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera

# any additional java opts you want to set. This will apply to both client and server operations
export ATLAS_OPTS="-Dlog4j.configuration=atlas-log4j.xml -Dsun.security.krb5.debug=true -Djava.security.krb5.conf=/etc/krb5.conf -Dzookeeper.sasl.client.username=zookeeper -Djava.security.auth.login.config=/DATA/disk1/apps/atlas/conf/atlas_jaas.conf"

export ATLAS_CONF=/DATA/disk1/apps/atlas/conf

# Where log files are stored. Defatult is logs directory under the base install location
export ATLAS_LOG_DIR=/var/log/atlas

# additional classpath entries
export ATLASCPPATH=

# data dir
export ATLAS_DATA_DIR=/DATA/disk1/apps/atlas/data

# pid dir
export ATLAS_PID_DIR=/var/run/atlas

# hbase conf dir
export HBASE_CONF_DIR=/etc/hbase/conf

# java heap size we want to set for the client. Default is 1024MB
#export ATLAS_CLIENT_HEAP=

# indicative values for large number of metadata entities (equal or more than 10,000s)
export ATLAS_SERVER_OPTS="-server -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+PrintTenuringDistribution -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$ATLAS_LOG_DIR/atlas_server.hprof -Xloggc:$ATLAS_LOG_DIR/gc-worker.log -verbose:gc -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1m -XX:+PrintGCDetails -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps"

# Where pid files are stored. Defatult is logs directory under the base install location
export ATLAS_PID_DIR=/var/run/atlas

# where the atlas titan db data is stored. Defatult is logs/data directory under the base install location
export ATLAS_DATA_DIR=/DATA/disk1/apps/atlas/data

# Where do you want to expand the war file. By Default it is in /server/webapp dir under the base install dir.
#export ATLAS_EXPANDED_WEBAPP_DIR=

# indicates whether or not a local instance of HBase should be started for Atlas
export MANAGE_LOCAL_HBASE=false

# indicates whether or not a local instance of Solr should be started for Atlas
export MANAGE_LOCAL_SOLR=false

# indicates whether or not cassandra is the embedded backend for Atlas
export MANAGE_EMBEDDED_CASSANDRA=false

# indicates whether or not a local instance of Elasticsearch should be started for Atlas
export MANAGE_LOCAL_ELASTICSEARCH=false

# Where do you want to expand the war file. By Default it is in /server/webapp dir under the base install dir
export ATLAS_EXPANDED_WEBAPP_DIR=/DATA/disk1/apps/atlas/server/webapp

export ATLAS_SERVER_HEAP="-Xms2048m -Xmx2048m -XX:MaxNewSize=600m -XX:MetaspaceSize=100m -XX:MaxMetaspaceSize=512m"

export MANAGE_LOCAL_HBASE=false

export MANAGE_LOCAL_SOLR=false

export MANAGE_EMBEDDED_CASSANDRA=false

export MANAGE_LOCAL_ELASTICSEARCH=false

```

### 2.2.3 vi $ATLAS_HOME/conf/atlas-application.properties
#### 2.2.3.1 vi $ATLAS_HOME/conf/atlas-application.properties # atlas 服务所在的服务器上配置类似如下：
```

atlas.audit.hbase.tablename=ATLAS_ENTITY_AUDIT_EVENTS
atlas.audit.hbase.zookeeper.quorum=onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com,onedts-dev-cdh-client-v01.smartparkos.com
atlas.audit.zookeeper.session.timeout.ms=60000
atlas.authentication.keytab=/etc/security/keytabs/atlas.service.keytab
atlas.authentication.principal=atlas/_HOST@SMARTPARKOS.COM
atlas.authentication.method.file=true
atlas.authentication.method.file.filename=/DATA/disk1/apps/atlas/conf/users-credentials.properties
atlas.authentication.method.kerberos=true
atlas.authentication.method.kerberos.keytab=/etc/security/keytabs/spnego.service.keytab
atlas.authentication.method.kerberos.principal=HTTP/_HOST@SMARTPARKOS.COM
atlas.authentication.method.kerberos.name.rules=RULE:[1:$1@$0](.*@SMARTPARKOS.COM)s/@.*//\
RULE:[2:$1@$0](atlas@SMARTPARKOS.COM)s/.*/atlas/\
RULE:[2:$1@$0](hdfs@SMARTPARKOS.COM)s/.*/hdfs/\
RULE:[2:$1@$0](hbase@SMARTPARKOS.COM)s/.*/hbase/\
RULE:[2:$1@$0](hive@SMARTPARKOS.COM)s/.*/hive/\
RULE:[2:$1@$0](mapred@SMARTPARKOS.COM)s/.*/mapred/\
RULE:[2:$1@$0](livy@SMARTPARKOS.COM)s/.*/livy/\
RULE:[2:$1@$0](yarn@SMARTPARKOS.COM)s/.*/yarn/\
RULE:[2:$1@$0](spark@SMARTPARKOS.COM)s/.*/spark/\
DEFAULT
atlas.authentication.method.ldap=true
atlas.authentication.method.ldap.ad.base.dn=
atlas.authentication.method.ldap.ad.bind.dn=
atlas.authentication.method.ldap.ad.bind.password=
atlas.authentication.method.ldap.ad.default.role=ROLE_USER
atlas.authentication.method.ldap.ad.domain=
atlas.authentication.method.ldap.ad.referral=ignore
atlas.authentication.method.ldap.ad.url=
atlas.authentication.method.ldap.ad.user.searchfilter=(sAMAccountName={0})
atlas.authentication.method.ldap.base.dn=dc=smartparkos,dc=com
atlas.authentication.method.ldap.bind.dn=uid=hadoopadmin,cn=users,cn=accounts,dc=smartparkos,dc=com
atlas.authentication.method.ldap.bind.password=Lscdh!654321111
atlas.authentication.method.ldap.default.role=ROLE_USER
atlas.authentication.method.ldap.groupRoleAttribute=cn
atlas.authentication.method.ldap.groupSearchBase=cn=groups,cn=accounts,dc=smartparkos,dc=com
# atlas.authentication.method.ldap.groupSearchFilter=(cn={0})
atlas.authentication.method.ldap.groupSearchFilter=(cn={0},cn=groups,cn=compat,dc=smartparkos,dc=com)
# atlas.authentication.method.ldap.groupSearchFilter=(cn=g_atlas)
# atlas.authentication.method.ldap.groupSearchFilter=(dn=cn={0},cn=groups,cn=compat,dc=smartparkos,dc=com)
atlas.authentication.method.ldap.referral=ignore
atlas.authentication.method.ldap.type=ldap
atlas.authentication.method.ldap.url=ldap://onedts-dev-cdh-ipa-v01.smartparkos.com:389
atlas.authentication.method.ldap.user.searchfilter=(uid={0})
atlas.authentication.method.ldap.userDNpattern=uid={0},cn=users,cn=accounts,dc=smartparkos,dc=com
# atlas.authorizer.impl=ranger
atlas.authorizer.impl=simple
atlas.authorizer.simple.authz.policy.file=atlas-simple-authz-policy.json
atlas.cluster.name=odtscluster
atlas.enableTLS=false
atlas.graph.index.search.solr.mode=cloud
atlas.graph.index.search.solr.wait-searcher=true
atlas.graph.index.search.solr.zookeeper-url=onedts-dev-cdh-client-v01.smartparkos.com:2181/solr,onedts-dev-cdh-master-v02.smartparkos.com:2181/solr,onedts-dev-cdh-master-v01.smartparkos.com:2181/solr
atlas.graph.storage.backend=hbase2
atlas.graph.storage.hbase.table=atlas_janus
atlas.graph.storage.hostname=onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com,onedts-dev-cdh-client-v01.smartparkos.com
atlas.jaas.KafkaClient.loginModuleControlFlag=required
atlas.jaas.KafkaClient.loginModuleName=com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.KafkaClient.option.keyTab=/etc/security/keytabs/atlas.service.keytab
atlas.jaas.KafkaClient.option.principal=atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.KafkaClient.option.serviceName=kafka
atlas.jaas.KafkaClient.option.storeKey=true
atlas.jaas.KafkaClient.option.useKeyTab=true
atlas.jaas.MyClient.0.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.MyClient.0.loginModuleControlFlag = required
atlas.jaas.MyClient.0.option.useKeyTab = true
atlas.jaas.MyClient.0.option.storeKey = true
atlas.jaas.MyClient.0.option.serviceName = kafka
atlas.jaas.MyClient.0.option.keyTab = /etc/security/keytabs/atlas.service.keytab
atlas.jaas.MyClient.0.option.principal = atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.MyClient.1.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.MyClient.1.loginModuleControlFlag = optional
atlas.jaas.MyClient.1.option.useKeyTab = true
atlas.jaas.MyClient.1.option.storeKey = true
atlas.jaas.MyClient.1.option.serviceName = kafka
atlas.jaas.MyClient.1.option.keyTab = /etc/security/keytabs/atlas.service.keytab
atlas.jaas.MyClient.1.option.principal = atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.ticketBased-KafkaClient.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.ticketBased-KafkaClient.loginModuleControlFlag = required
atlas.jaas.ticketBased-KafkaClient.option.useTicketCache = true
atlas.jaas.ticketBased-KafkaClient.option.renewTicket = true
atlas.jaas.ticketBased-KafkaClient.option.serviceName = kafka
atlas.kafka.auto.commit.enable=false
atlas.kafka.bootstrap.servers=onedts-dev-cdh-datanode-v02.smartparkos.com:9092,onedts-dev-cdh-datanode-v03.smartparkos.com:9092,onedts-dev-cdh-datanode-v01.smartparkos.com:9092
atlas.kafka.hook.group.id=atlas
atlas.kafka.entities.group.id=atlas_entities
atlas.kafka.sasl.kerberos.service.name=kafka
atlas.kafka.security.protocol=SASL_PLAINTEXT
atlas.kafka.sasl.mechanism=GSSAPI
atlas.kafka.zookeeper.connect=onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181
atlas.kafka.zookeeper.connection.timeout.ms=30000
atlas.kafka.zookeeper.session.timeout.ms=60000
atlas.kafka.zookeeper.sync.time.ms=20
atlas.lineage.schema.query.hive_table=hive_table where __guid='%s'\, columns
atlas.lineage.schema.query.Table=Table where __guid='%s'\, columns
atlas.notification.create.topics=true
atlas.notification.embedded=false
atlas.notification.replicas=1
atlas.notification.topics=ATLAS_HOOK,ATLAS_ENTITIES
atlas.proxyusers=
atlas.rest.address=http://onedts-dev-cdh-client-v01.smartparkos.com:21099,http://onedts-dev-cdh-client-v02.smartparkos.com:21099
atlas.server.address.id1=onedts-dev-cdh-client-v01.smartparkos.com:21099
atlas.server.address.id2=onedts-dev-cdh-client-v02.smartparkos.com:21099
atlas.server.bind.address=0.0.0.0
atlas.server.ha.enabled=true
atlas.client.ha.retries=4
# atlas.server.ha.zookeeper.acl=auth:
atlas.server.ha.zookeeper.connect=onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181
# atlas.server.ha.zookeeper.zkroot=/apache_atlas
atlas.server.http.port=21099
atlas.server.https.port=21443
atlas.server.ids=id1,id2
atlas.simple.authz.policy.file=/DATA/disk1/apps/atlas/conf/atlas-simple-authz-policy.json
atlas.solr.kerberos.enable=true
atlas.ssl.exclude.protocols=TLSv1.2
atlas.sso.knox.browser.useragent=
atlas.sso.knox.enabled=false
atlas.sso.knox.providerurl=
atlas.sso.knox.publicKey=

```

#### 注意1：atlas 服务 运行在 atlas 用户下
#### 注意2：相关 kerberos 票据文件生成类似如下，分别在 onedts-dev-cdh-client-v01 & onedts-dev-cdh-client-v02 服务器操作
```
kinit admin  # 输入 admin 的密码

# atlas 用户创建，只需操作一次即可
atlas_user=atlas
ipa user-add ${atlas_user} --first=${atlas_user} --last=${atlas_user} --shell=/bin/bash --homedir=/home/${atlas_user}

# atlas 用户票据生成， 只需在其中一台服务器操作，然后 copy 到另外一台服务器相同目录（ 注意文件权限 )
ipa service-add ${atlas_user}/$(hostname -f)@SMARTPARKOS.COM
ipa_server=onedts-dev-cdh-ipa-v01.smartparkos.com
user_home=/DATA/disk1/home/${atlas_user};
mkdir -p ${user_home}/security/keytabs/;
ipa-getkeytab -s ${ipa_server} \
-p  ${atlas_user} \
-e aes256-cts-hmac-sha1-96,aes128-cts-hmac-sha1-96,aes256-cts-hmac-sha384-192,aes128-cts-hmac-sha256-128,des3-cbc-sha1,arcfour-hmac \
-k ${user_home}/security/keytabs/${user_name}.keytab \
-P atlas\#111222

chown -R ${atlas_user}:${atlas_user} ${user_home};
chmod 400 ${user_home}/security/keytabs/${user_name}.keytab;

# atlas 服务票据，分别在 onedts-dev-cdh-client-v01 & onedts-dev-cdh-client-v02 服务器操作
ipa-getkeytab -s ${ipa_server} \
-p  ${atlas_user}/$(hostname -f)@SMARTPARKOS.COM \
-e aes256-cts-hmac-sha1-96,aes128-cts-hmac-sha1-96,aes256-cts-hmac-sha384-192,aes128-cts-hmac-sha256-128,des3-cbc-sha1,arcfour-hmac \
-k /etc/security/keytabs/atlas.service.keytab \

chown atlas:hadoop  /etc/security/keytabs/atlas.service.keytab;
chmod 440 /etc/security/keytabs/atlas.service.keytab;

```

#### 注意3：创建 Kafka topic ，并授权( 如果配置了 Sentry for kafka 权限控制的话 )
```
# 1. Create Kafka Topic 
kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-master-v01.smartparkos.com:2181 \
--create --replication-factor 3 \
--partitions 3 \
--topic ATLAS_ENTITIES

kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-master-v01.smartparkos.com:2181 \
--create --replication-factor 3 \
--partitions 3 \
--topic ATLAS_HOOK

# 2. View topic
kafka-topics --list \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-master-v01.smartparkos.com:2181

kafka-topics --describe --zookeeper localhost:2181 --topic ATLAS_HOOK
kafka-topics --describe --zookeeper localhost:2181 --topic ATLAS_ENTITIES

# 3. create znode ( with hadoop user )
cd /opt/cloudera/parcels/CDH/lib/zookeeper/bin/;
./zkCli.sh

create /apache_atlas_v1 apache_atlas_v1 
getAcl /apache_atlas_v1

create /solr solr
getAcl /solr


# 4. Sentry 授权 略

```

#### 注意4：创建 solr 索引
```
cd /DATA/disk1/apps/atlas/;
solrctl instancedir --create atlas conf/solr/
solrctl collection --create vertex_index -s 3 -c atlas -r 1
solrctl collection --create edge_index -s 3 -c atlas -r 1
solrctl collection --create fulltext_index -s 3 -c atlas -r 1
```

#### 2.2.3.2 vi $ATLAS_HOME/conf/atlas-application.properties # HiveServer2服务所在的服务器上配置类似如下( 即 我的服务器是 onedts-dev-cdh-master-v01 & onedts-dev-cdh-master-v02 )
```

atlas.authentication.method.kerberos=True
atlas.cluster.name=odtscluster
atlas.hook.hive.keepAliveTime=10
atlas.hook.hive.maxThreads=5
atlas.hook.hive.minThreads=5
atlas.hook.hive.numRetries=3
atlas.hook.hive.queueSize=1000
atlas.hook.hive.synchronous=false
atlas.jaas.KafkaClient.loginModuleControlFlag=required
atlas.jaas.KafkaClient.loginModuleName=com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.KafkaClient.option.keyTab=/etc/security/keytabs/hive.service.keytab
atlas.jaas.KafkaClient.option.principal=hive/_HOST@SMARTPARKOS.COM
atlas.jaas.KafkaClient.option.serviceName=kafka
atlas.jaas.KafkaClient.option.storeKey=True
atlas.jaas.KafkaClient.option.useKeyTab=True
atlas.jaas.ticketBased-KafkaClient.loginModuleControlFlag=required
atlas.jaas.ticketBased-KafkaClient.loginModuleName=com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.ticketBased-KafkaClient.option.useTicketCache=true
atlas.kafka.bootstrap.servers=onedts-dev-cdh-datanode-v02.smartparkos.com:9092,onedts-dev-cdh-datanode-v03.smartparkos.com:9092,onedts-dev-cdh-datanode-v01.smartparkos.com:9092
atlas.kafka.hook.group.id=atlas
atlas.kafka.sasl.kerberos.service.name=kafka
atlas.kafka.security.protocol=SASL_PLAINTEXT
atlas.kafka.zookeeper.connect=onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181
atlas.kafka.zookeeper.connection.timeout.ms=30000
atlas.kafka.zookeeper.session.timeout.ms=60000
atlas.kafka.zookeeper.sync.time.ms=20
atlas.notification.create.topics=True
atlas.notification.replicas=1
atlas.notification.topics=ATLAS_HOOK,ATLAS_ENTITIES
atlas.rest.address=http://onedts-dev-cdh-client-v01.smartparkos.com:21099,http://onedts-dev-cdh-client-v02.smartparkos.com:21099

```

### 注意1. /etc/security/keytabs/hive.service.keytab 文件是根据 HiveServer2 服务 现有的 hive.keytab 文件 copy 而来的 (执行如下命令，将找到很多 hive.keytab 文件，copy 最近的一个文件，验证其票据是OK的，即可)，操作类似如下。如果你有多个 "HiveServer2 服务" 运行在多台服务器，记得每台服务器都同样操作一下。
### 注意2. 如果 重新生成了 kerberos 票据的话，将导致原来的票据文件失效，需要重新执行类似如下操作生成新的票据文件
```
[root@onedts-dev-cdh-master-v01 conf]# find / -name hive*.keytab|grep HIVESERVER2
find: ‘/proc/31626’: 没有那个文件或目录
/run/cloudera-scm-agent/process/9968-hive-HIVESERVER2/hive.keytab
/run/cloudera-scm-agent/process/9964-hive-HIVESERVER2/hive.keytab
...

[root@onedts-dev-cdh-master-v01 conf]# klist -kt /run/cloudera-scm-agent/process/9968-hive-HIVESERVER2/hive.keytab
Keytab name: FILE:/run/cloudera-scm-agent/process/9968-hive-HIVESERVER2/hive.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   9 2021-07-08T18:00:16 HTTP/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
   6 2021-07-08T18:00:16 hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM

[root@onedts-dev-cdh-master-v01 conf]# kinit -kt /run/cloudera-scm-agent/process/9968-hive-HIVESERVER2/hive.keytab hive/onedts-dev-cdh-master-v01.smartparkos.com

[root@onedts-dev-cdh-master-v01 conf]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: hive/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM

Valid starting       Expires              Service principal
2021-07-12T18:31:41  2021-07-13T18:31:41  krbtgt/SMARTPARKOS.COM@SMARTPARKOS.COM
	renew until 2021-07-19T18:31:41

cp /run/cloudera-scm-agent/process/9968-hive-HIVESERVER2/hive.keytab /etc/security/keytabs/hive.service.keytab;
chown hive:hadoop /etc/security/keytabs/hive.service.keytab;
chmod 440 /etc/security/keytabs/hive.service.keytab;

```

### 2.2.4 vi $ATLAS_HOME/conf/atlas_jaas.conf
### 注意： 因为 "_HOST" 变量在 jaas 配置文件中不支持，所以在不同的服务器，请将配置文件中的 "_HOST" 替换为 相关的主机名，比如 "onedts-dev-cdh-client-v01.smartparkos.com"
```

KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  storeKey=true
  serviceName=kafka
  keyTab="/etc/security/keytabs/atlas.service.keytab"
  principal="atlas/_HOST@SMARTPARKOS.COM";
};

Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  storeKey=true
  serviceName=zookeeper
  keyTab="/etc/security/keytabs/atlas.service.keytab"
  principal="atlas/_HOST@SMARTPARKOS.COM";
};

MyClient.0 {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  storeKey=true
  serviceName=kafka
  keyTab="/etc/security/keytabs/atlas.service.keytab"
  principal="atlas/_HOST@SMARTPARKOS.COM";
};

MyClient.1 {
  com.sun.security.auth.module.Krb5LoginModule optional
  useKeyTab=true
  storeKey=true
  serviceName=kafka
  keyTab="/etc/security/keytabs/atlas.service.keytab"
  principal="atlas/_HOST@SMARTPARKOS.COM";
};

ticketBased-KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=true
  renewTicket=true
  serviceName="kafka";
};

```

### 2.2.5 vi $ATLAS_HOME/atlas-jaas.properties
```

#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
#
#########  Notification Configs  #########
atlas.notification.embedded=true

atlas.kafka.zookeeper.connect=onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181
atlas.kafka.bootstrap.servers=onedts-dev-cdh-datanode-v01.smartparkos.com:9092,onedts-dev-cdh-datanode-v02.smartparkos.com:9092,onedts-dev-cdh-datanode-v03.smartparkos.com:9092
atlas.kafka.data=${sys:atlas.data}/kafka
atlas.kafka.zookeeper.session.timeout.ms=4000
atlas.kafka.zookeeper.sync.time.ms=20
atlas.kafka.consumer.timeout.ms=100
atlas.kafka.auto.commit.interval.ms=100
atlas.kafka.hook.group.id=atlas
atlas.kafka.entities.group.id=atlas_entities
atlas.kafka.auto.commit.enable=false

######## JAAS configs ##################

atlas.jaas.KafkaClient.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.KafkaClient.loginModuleControlFlag = required
atlas.jaas.KafkaClient.option.useKeyTab = true
atlas.jaas.KafkaClient.option.storeKey = true
atlas.jaas.KafkaClient.option.serviceName = "kafka"
atlas.jaas.KafkaClient.option.keyTab = /etc/security/keytabs/atlas.keytab
atlas.jaas.KafkaClient.option.principal = atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.MyClient.0.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.MyClient.0.loginModuleControlFlag = required
atlas.jaas.MyClient.0.option.useKeyTab = true
atlas.jaas.MyClient.0.option.storeKey = true
atlas.jaas.MyClient.0.option.serviceName = kafka
atlas.jaas.MyClient.0.option.keyTab = /etc/security/keytabs/atlas.service.keytab
atlas.jaas.MyClient.0.option.principal = atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.MyClient.1.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.MyClient.1.loginModuleControlFlag = optional
atlas.jaas.MyClient.1.option.useKeyTab = true
atlas.jaas.MyClient.1.option.storeKey = true
atlas.jaas.MyClient.1.option.serviceName = kafka
atlas.jaas.MyClient.1.option.keyTab = /etc/security/keytabs/atlas.service.keytab
atlas.jaas.MyClient.1.option.principal = atlas/_HOST@SMARTPARKOS.COM
atlas.jaas.ticketBased-KafkaClient.loginModuleName = com.sun.security.auth.module.Krb5LoginModule
atlas.jaas.ticketBased-KafkaClient.loginModuleControlFlag = required
atlas.jaas.ticketBased-KafkaClient.option.useTicketCache = true
atlas.jaas.ticketBased-KafkaClient.option.renewTicket = true
atlas.jaas.ticketBased-KafkaClient.option.serviceName = kafka

```

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 3. 配置 atlas hook

# -------------------------------------------------------------------
## 3.1 hive hook
## 在 Cloudera manager 管理 hive 配置页做如下配置

### 3.1.1 "hive-site.xml 的 Hive 服务高级配置代码段（安全阀）"
```
hive.exec.post.hooks=org.apache.atlas.hive.hook.HiveHook

```

### 3.1.2 "hive-site.xml 的 Hive 客户端高级配置代码段（安全阀）"
```
hive.exec.post.hooks=org.apache.atlas.hive.hook.HiveHook
atlas.rest.address=http://onedts-dev-cdh-client-v01.smartparkos.com:21099,http://onedts-dev-cdh-client-v01.smartparkos.com:21099

```

### 3.1.3 "hive-site.xml 的 HiveServer2 高级配置代码段（安全阀）"
```
hive.exec.post.hooks=org.apache.atlas.hive.hook.HiveHook
hive.reloadable.aux.jars.path=/DATA/disk1/apps/atlas/hook/hive

```

### 3.1.4 "Hive 辅助 JAR 目录"
```
/DATA/disk1/apps/atlas/hook/hive

```

### 3.1.5 "hive-env.sh 的 Gateway 客户端环境高级配置代码段（安全阀）"
```
HIVE_AUX_JARS_PATH=/DATA/disk1/apps/atlas/hook/hive

```

### 3.1.6 "HiveServer2 环境高级配置代码段（安全阀）"
```
HIVE_AUX_JARS_PATH=/DATA/disk1/apps/atlas/hook/hive

```

### 3.1.7 配置完后，保存，重启并测试

# -------------------------------------------------------------------
## 3.2 impala hook # 未测试

# -------------------------------------------------------------------
# 4. 安装操作命令

```

# mkdir -p /DATA/disk1/apps/;
cd /DATA/disk1/apps/;
rm -rf apache-atlas-2.1.0;
tar -xvf apache-atlas-2.1.0-bin.tar.gz;
unlink atlas; ln -s ./apache-atlas-2.1.0 ./atlas;

cd apache-atlas-2.1.0;
rm -rf conf;
cp ~/conf.tar.gz ./;
tar -xvf conf.tar.gz;
rm -rf conf.tar.gz;
cd /DATA/disk1/apps/atlas/conf/;
\cp ~/atlas-application.properties ./;
\cp ~/atlas_jaas.conf ./;

cd /DATA/disk1/apps/atlas/hook/hive/;
\cp ../../conf/atlas-application.properties ./;
zip -u atlas-plugin-classloader-2.1.0.jar atlas-application.properties
chown -R atlas:atlas /DATA/disk1/apps/apache-atlas-2.1.0;

mkdir -p /DATA/disk1/apps/atlas/data/kafka;
chown -R atlas:atlas /DATA/disk1/apps/atlas/data;

cd /etc/hive/conf/;\cp /DATA/disk1/apps/atlas/conf/atlas-application.properties ./; \cp /DATA/disk1/apps/atlas/conf/atlas-env.sh ./;ls -l ./


# setAcl
cd /opt/cloudera/parcels/CDH/lib/zookeeper/;
./bin/zkCli.sh
setAcl /zookeeper world:anyone:cdrwa,sasl:atlas:cdwra,sasl:bigdata:cdwra,sasl:hive:cdwra
setAcl /brokers world:anyone:cdrwa,sasl:atlas:cdwra,sasl:bigdata:cdwra,sasl:hive:cdwra

create /apache_atlas210_v1 apache_atlas210_v1
getAcl /apache_atlas210_v1

setAcl /apache_atlas210_v1 world:anyone:cdrwa,sasl:atlas@SMARTPARKOS.COM:cdrwa

# 启动 export JVMFLAGS="-Djava.security.auth.login.config=/DATA/disk1/apps/atlas/conf/atlas_jaas.conf"
jps|grep Atlas|awk '{print $1}'|xargs kill -9
cd /DATA/disk1/apps/apache-atlas-2.1.0/;
rm -rf ./logs/*;
mkdir ./logs/;
rm -rf /var/log/atlas/*;
# touch logs/gc-worker.log;
# ./bin/quick_start.py
./bin/atlas_start.py
./bin/atlas_admin.py -status

# -------------------
# 查看日志 
ssh -p36565 root@onedts-dev-cdh-master-v02
tail -n 1000 /var/log/hive/hadoop-cmf-hive-HIVESERVER2-$(hostname -f).log.out

ssh -p36565 root@onedts-dev-cdh-master-v01
tail -n 1000 /var/log/hive/hadoop-cmf-hive-HIVESERVER2-$(hostname -f).log.out


# 测试 hive hook
export HIVE_HOME=/opt/cloudera/parcels/CDH/lib/hive;
export HIVE_CONF_DIR=/etc/hive/conf;
cd /DATA/disk1/apps/atlas/;
./bin/import-hive.sh \
-Djavax.net.ssl.trustStore=/DATA/disk1/apps/atlas/atlas.keystore \
-Djavax.net.ssl.trustStorePassword=bee56915 \
-Dsun.security.jgss.debug=true \
-Djavax.security.auth.useSubjectCredsOnly=false \
-Djava.security.krb5.conf=/etc/krb5.conf \
-Djava.security.auth.login.config=/DATA/disk1/apps/atlas/conf/atlas_jaas.conf 

export HIVE_HOME=/opt/cloudera/parcels/CDH/lib/hive;
export HIVE_CONF_DIR=/etc/hive/conf;
cd /DATA/disk1/apps/atlas/;
./bin/import-hive.sh \
-Dsun.security.jgss.debug=true \
-Djavax.security.auth.useSubjectCredsOnly=false \
-Djava.security.krb5.conf=/etc/krb5.conf \
-Djava.security.auth.login.config=/DATA/disk1/apps/atlas/conf/atlas_jaas.conf 

# -------------------
# 查看日志 
ssh -p36565 root@onedts-dev-cdh-master-v02
tail -n 1000 /var/log/hive/hadoop-cmf-hive-HIVESERVER2-$(hostname -f).log.out

ssh -p36565 root@onedts-dev-cdh-master-v01
tail -n 1000 /var/log/hive/hadoop-cmf-hive-HIVESERVER2-$(hostname -f).log.out


```

# -------------------------------------------------------------------
# 5. 查看 atlas 高可用状态
## 5.1 用 ./bin/atlas_admin.py 查看
```
cd /DATA/disk1/apps/atlas/;
./bin/atlas_admin.py -u admin:Bee\#56915lym -status

# 我的输出类似如下

[atlas@onedts-dev-cdh-client-v01 atlas]$ cd /DATA/disk1/apps/atlas/;
[atlas@onedts-dev-cdh-client-v01 atlas]$ ./bin/atlas_admin.py -u admin:Bee\#56915lym -status
ACTIVE

[atlas@onedts-dev-cdh-client-v02 atlas]$ cd /DATA/disk1/apps/atlas/;
[atlas@onedts-dev-cdh-client-v02 atlas]$ ./bin/atlas_admin.py -u admin:Bee\#56915lym -status
ACTIVE

```

## 5.2 用 curl 命令查看
```

curl -u luoyoumou:bee\#56915 -X GET http://onedts-dev-cdh-client-v01.smartparkos.com:21099/api/atlas/admin/status
curl -u luoyoumou:bee\#56915 -X GET http://onedts-dev-cdh-client-v02.smartparkos.com:21099/api/atlas/admin/status

# curl 好像不提供用户名/密码 也可以
curl -X GET http://onedts-dev-cdh-client-v01.smartparkos.com:21099/api/atlas/admin/status
curl -X GET http://onedts-dev-cdh-client-v02.smartparkos.com:21099/api/atlas/admin/status

# 我的输出类似如下
[atlas@onedts-dev-cdh-client-v02 atlas]$ curl -u luoyoumou:bee\#56915 -X GET http://onedts-dev-cdh-client-v01.smartparkos.com:21099/api/atlas/admin/status; echo '';
{"Status":"PASSIVE"}
[atlas@onedts-dev-cdh-client-v02 atlas]$ curl -u luoyoumou:bee\#56915 -X GET http://onedts-dev-cdh-client-v02.smartparkos.com:21099/api/atlas/admin/status; echo '';
{"Status":"ACTIVE"}

```

# -------------------------------------------------------------------
# 6. 测试过程中，清理数据相关操作
## 6.1 kafka topic
```
su - hadoop
kafka-topics --list --zookeeper onedts-dev-cdh-client-v01.smartparkos.com:2181
kafka-topics --zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181 --delete --topic ATLAS_HOOK
kafka-topics --zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181 --delete --topic ATLAS_ENTITIES


kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-master-v01.smartparkos.com:2181 \
--create --replication-factor 2 \
--partitions 1 \
--topic ATLAS_ENTITIES

kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-master-v01.smartparkos.com:2181 \
--create --replication-factor 2 \
--partitions 1 \
--topic ATLAS_HOOK

```

## 6.2 hbase
```
su - hadoop
hbase shell
list
disable 'ATLAS_ENTITY_AUDIT_EVENTS'
disable 'atlas_janus'
drop 'ATLAS_ENTITY_AUDIT_EVENTS'
drop 'atlas_janus'

hbase shell
grant 'atlas', 'RWXCA', '@default'
grant 'hive', 'RWXCA', '@default'

grant 'atlas', 'RWXCA', 'default:ATLAS_ENTITY_AUDIT_EVENTS'
grant 'atlas', 'RWXCA', 'default:atlas_janus'

grant 'hive', 'RWXCA', 'default:ATLAS_ENTITY_AUDIT_EVENTS'
grant 'hive', 'RWXCA', 'default:atlas_janus'

```

## 6.3 solr
```
# 删除

solrctl collection --delete vertex_index
solrctl collection --delete edge_index
solrctl collection --delete fulltext_index

solrctl instancedir --delete atlas

# 创建
cd /DATA/disk1/apps/atlas/;
solrctl instancedir --create atlas conf/solr/
solrctl collection --create vertex_index -s 3 -c atlas -r 1
solrctl collection --create edge_index -s 3 -c atlas -r 1
solrctl collection --create fulltext_index -s 3 -c atlas -r 1

```

# -------------------------------------------------------------------
# 7. Atlas Haproxy 代理配置
## 参考：https://atlas.apache.org/1.2.0/HighAvailability.html
##      https://serverfault.com/questions/801247/haproxy-httpchk-with-authentication
## 不提供用户名密码认证也可以，故 "7.1" 操作可以忽略

## 7.1 执行命令：echo -n "luoyoumou:bee#56915" | base64 
```
bHVveW91bW91OmJlZSM1NjkxNQ==

```

## 7.2 
```
#---------------------------------------------------------------------
# Atlas HA
#---------------------------------------------------------------------
frontend atlas_fe
  bind *:21099
  default_backend atlas_be

backend atlas_be
  mode http
  option httpchk get /api/atlas/admin/status
  http-check expect string ACTIVE
  balance roundrobin
  server atlas1 onedts-dev-cdh-client-v01.smartparkos.com:21099 check
  server atlas2 onedts-dev-cdh-client-v02.smartparkos.com:21099 check

```

## 7.3 配置好 haproxy 后，重启，测试
```
systemctl restart haproxy
systemctl status haproxy

curl -X GET http://onedts-dev-cdh-cm-v01.smartparkos.com:21099/api/atlas/admin/status

# 我的输出类似如下
[root@onedts-dev-cdh-cm-v01 ~]# systemctl restart haproxy
[root@onedts-dev-cdh-cm-v01 ~]# systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since 二 2021-07-20 10:03:39 CST; 3ms ago
 Main PID: 3965 (haproxy-systemd)
    Tasks: 2
   CGroup: /system.slice/haproxy.service
           ├─3965 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           └─3967 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

7月 20 10:03:39 onedts-dev-cdh-cm-v01 systemd[1]: Started HAProxy Load Balancer.
7月 20 10:03:39 onedts-dev-cdh-cm-v01 haproxy-systemd-wrapper[3965]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
[root@onedts-dev-cdh-cm-v01 ~]#
[root@onedts-dev-cdh-cm-v01 ~]# curl -X GET http://onedts-dev-cdh-cm-v01.smartparkos.com:21099/api/atlas/admin/status
{"Status":"ACTIVE"}

```

# -------------------------------------------------------------------
# 8. Atlas sqoop hook 配置 & 测试
# 参考：https://atlas.apache.org/1.2.0/Hook-Sqoop.html

# 8.1 安装前 sqoop 测试
```

```

# 8.2 配置 sqoop hook： 在 sqoop-site.xml 配置文件 ( 即 "sqoop-conf/sqoop-site.xml 的 Sqoop 1 Client 客户端高级配置代码段（安全阀）" ) 添加如下配置
```

sqoop.job.data.publish.class=org.apache.atlas.sqoop.hook.SqoopHook

```

# 8.3 Link <atlas package>/hook/sqoop/*.jar in sqoop lib
```
ls -l /DATA/disk1/apps/atlas/hook/sqoop/;

[root@onedts-dev-cdh-client-v01 sqoop]# ls -l
总用量 44
-rw-r--r-- 1 atlas atlas 17499 7月  21 20:42 atlas-plugin-classloader-2.1.0.jar
drwxr-xr-x 2 atlas atlas  4096 7月  21 20:42 atlas-sqoop-plugin-impl
-rw-r--r-- 1 atlas atlas 17335 7月  21 20:42 sqoop-bridge-shim-2.1.0.jar

cd /opt/cloudera/parcels/CDH/lib/sqoop/lib/;
ln -s /DATA/disk1/apps/atlas/hook/sqoop/atlas-plugin-classloader-2.1.0.jar ./atlas-plugin-classloader-2.1.0.jar;
ln -s /DATA/disk1/apps/atlas/hook/sqoop/sqoop-bridge-shim-2.1.0.jar ./sqoop-bridge-shim-2.1.0.jar;

```

# 8.4 
```
\cp /DATA/disk1/apps/atlas/hook/sqoop/atlas-sqoop-plugin-impl/*.jar /opt/cloudera/parcels/CDH/lib/sqoop/lib/;
```
# 8.5 重启 sqoop 后，同步配置文件
```

cd /etc/sqoop/conf/;\cp /DATA/disk1/apps/atlas/conf/atlas-application.properties ./; \cp /DATA/disk1/apps/atlas/conf/atlas-env.sh ./;ls -l ./

```

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 9. 报错解决：
## 9.1 报错如下
```
21/07/24 23:32:58 INFO mapreduce.ImportJobBase: Publishing Hive/Hcat import job data to Listeners for table null
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/configuration/PropertiesConfiguration
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
	at java.net.URLClassLoader.defineClass(URLClassLoader.java:467)
	at java.net.URLClassLoader.access$100(URLClassLoader.java:73)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:368)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	at org.apache.atlas.hook.AtlasHook.<clinit>(AtlasHook.java:80)
	at org.apache.atlas.sqoop.hook.SqoopHook.<clinit>(SqoopHook.java:82)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:264)

```

## 解决：下载 commons-configuration-1.10.jar 到 /opt/cloudera/parcels/CDH/lib/sqoop/lib/
```
cd /opt/cloudera/parcels/CDH/lib/sqoop/lib/;
wget https://repo1.maven.org/maven2/commons-configuration/commons-configuration/1.10/commons-configuration-1.10.jar

```

