# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 作者：luoyoumou 钉钉群：30931114 ( 万峰大数据技术交流 )

# DownLoad Url: https://griffin.apache.org/docs/download.html

# 参考： https://griffin.apache.org/docs/quickstart-cn.html
#       https://github.com/apache/griffin/blob/master/griffin-doc/deploy/deploy-guide.md
#       https://github.com/apache/griffin
#       https://blog.csdn.net/u010834071/article/details/109843258
#       https://www.zxmseed.com/blog/220167313
#       http://www.cxyzjd.com/article/u010834071/109989924
#       https://blog.csdn.net/qq_42168543/article/details/110667683

# 注意：在 onedts-dev-cdh-client-v01 服务器编译 & 安装 griffin
# 注意：大数据环境是 cdh6.3.2 for kerberos(FreeIPA) 环境(配置了 hive & kafka for sentry 权限控制 )

# Integrating Apache Griffin to CDH Clusters

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 一. 编译 & 安装 # 在 onedts-dev-cdh-client-v01 服务器执行
## 1.1 下载相关软件
```
mkdir -p /DATA/disk1/software/griffin/;
cd /DATA/disk1/software/griffin/;
wget https://archive.apache.org/dist/griffin/0.6.0/griffin-0.6.0-source-release.zip

# 最新的 git 版本： git clone https://github.com/apache/griffin.git

```

# -------------------------------------------------------------------
## 1.2 备份 并修改相关配置文件 # 在 onedts-dev-cdh-client-v01 服务器执行
```
conf_dir=conf_bak_060;
mkdir -p /DATA/disk1/software/griffin/${conf_dir}/;
cd /DATA/disk1/software/griffin/${conf_dir}/;
\cp ../griffin-0.6.0/service/pom.xml ./;
\cp ../griffin-0.6.0/service/hibernate_mysql_pom.xml ./;
\cp ../griffin-0.6.0/service/src/main/resources/application.properties ./;
\cp ../griffin-0.6.0/service/src/main/resources/quartz.properties ./;
\cp ../griffin-0.6.0/service/src/main/resources/sparkProperties.json ./;
\cp ../griffin-0.6.0/service/src/main/resources/env/env_batch.json ./;
\cp ../griffin-0.6.0/service/src/main/resources/env/env_streaming.json ./;
\cp ../griffin-0.6.0/ui/angular/src/assets/img/logo.png ./;
\cp ../griffin-0.6.0/ui/angular/src/index.html ./;
\cp ../griffin-0.6.0/ui/angular/src/test.html ./;
\cp ../griffin-0.6.0/service/src/main/java/org/apache/griffin/core/metastore/hive/HiveMetaStoreServiceJdbcImpl.java ./;
\cp ../griffin-0.6.0/service/src/main/java/org/apache/griffin/core/metastore/hive/HiveMetaStoreProxy.java ./;
\cp ../griffin-0.6.0/service/src/test/java/org/apache/griffin/core/metastore/hive/HiveMetastoreServiceJDBCImplTest.java ./;

```

### 1.2.1 vi ./${conf_dir}/HiveMetaStoreServiceJdbcImpl.java
```
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package org.apache.griffin.core.metastore.hive;

import java.io.IOException;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import javax.annotation.PostConstruct;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hive.metastore.api.FieldSchema;
import org.apache.hadoop.hive.metastore.api.StorageDescriptor;
import org.apache.hadoop.hive.metastore.api.Table;
import org.apache.hadoop.security.UserGroupInformation;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;


@Service
@Qualifier(value = "jdbcSvc")
@CacheConfig(cacheNames = "jdbcHive", keyGenerator = "cacheKeyGenerator")
public class HiveMetaStoreServiceJdbcImpl implements HiveMetaStoreService {

    private static final Logger LOGGER = LoggerFactory
        .getLogger(HiveMetaStoreService.class);

    private static final String SHOW_TABLES_IN = "show tables in ";

    private static final String SHOW_DATABASE = "show databases";

    private static final String SHOW_CREATE_TABLE = "show create table ";

    @Value("${hive.jdbc.className}")
    private String hiveClassName;

    @Value("${hive.jdbc.url}")
    private String hiveUrl;

    @Value("${hive.need.kerberos}")
    private String needKerberos;

    @Value("${hive.keytab.user}")
    private String keytabUser;

    @Value("${hive.keytab.path}")
    private String keytabPath;

    @Value("${hive.krb5.path}")
    private String hiveKrb5Path;

    private Connection conn;

    public void setConn(Connection conn) {
        this.conn = conn;
    }

    public void setHiveClassName(String hiveClassName) {
        this.hiveClassName = hiveClassName;
    }

    public void setNeedKerberos(String needKerberos) {
        this.needKerberos = needKerberos;
    }

    public void setKeytabUser(String keytabUser) {
        this.keytabUser = keytabUser;
    }

    public void setKeytabPath(String keytabPath) {
        this.keytabPath = keytabPath;
    }

    public void setHiveKrb5Path(String hiveKrb5Path) {
        this.hiveKrb5Path = hiveKrb5Path;
    }

    @PostConstruct
    public void init() {
        LOGGER.info("HiveConfigure: {} {} {} {}",hiveClassName,keytabUser,keytabPath,hiveKrb5Path );

        if (needKerberos != null && needKerberos.equalsIgnoreCase("true")) {
            LOGGER.info("Hive need Kerberos Auth.");

            System.setProperty("java.security.krb5.conf", hiveKrb5Path);
            Configuration conf = new Configuration();
            conf.set("hadoop.security.authentication", "Kerberos");
            UserGroupInformation.setConfiguration(conf);
            try {
                UserGroupInformation.loginUserFromKeytab(keytabUser, keytabPath);
            } catch (IOException e) {
                LOGGER.error("Register Kerberos has error. {}", e.getMessage());
            }
        }
    }

    @Override
    @Cacheable(unless = "#result==null")
    public Iterable<String> getAllDatabases() {
        return queryHiveString(SHOW_DATABASE);
    }

    @Override
    @Cacheable(unless = "#result==null")
    public Iterable<String> getAllTableNames(String dbName) {
        return queryHiveString(SHOW_TABLES_IN + dbName);
    }

    @Override
    @Cacheable(unless = "#result==null")
    public Map<String, List<String>> getAllTableNames() {
        // If there has a lots of databases in Hive, this method will lead to Griffin crash
        Map<String, List<String>> res = new HashMap<>();
        for (String dbName : getAllDatabases()) {
            List<String> list = (List<String>) queryHiveString(SHOW_TABLES_IN + dbName);
            res.put(dbName, list);
        }
        return res;
    }

    @Override
    public List<Table> getAllTable(String db) {
        return null;
    }

    @Override
    public Map<String, List<Table>> getAllTable() {
        return null;
    }

    @Override
    @Cacheable(unless = "#result==null")
    public Table getTable(String dbName, String tableName) {
        Table result = new Table();
        result.setDbName(dbName);
        result.setTableName(tableName);

        String sql = SHOW_CREATE_TABLE + dbName + "." + tableName;
        Statement stmt = null;
        ResultSet rs = null;
        StringBuilder sb = new StringBuilder();

        try {
            Class.forName(hiveClassName);
            if (conn == null) {
                conn = DriverManager.getConnection(hiveUrl);
            }
            LOGGER.info("got connection");

            stmt = conn.createStatement();
            rs = stmt.executeQuery(sql);
            while (rs.next()) {
                String s = rs.getString(1);
                sb.append(s);
            }
            String location = getLocation(sb.toString());
            List<FieldSchema> cols = getColums(sb.toString());
            StorageDescriptor sd = new StorageDescriptor();
            sd.setLocation(location);
            sd.setCols(cols);
            result.setSd(sd);
        } catch (Exception e) {
            LOGGER.error("Query Hive Table metadata has error. {}", e.getMessage());
        } finally {
            closeConnection(stmt, rs);
        }
        return result;
    }

    @Scheduled(fixedRateString =
        "${cache.evict.hive.fixedRate.in.milliseconds}")
    @CacheEvict(
        cacheNames = "jdbcHive",
        allEntries = true,
        beforeInvocation = true)
    public void evictHiveCache() {
        LOGGER.info("Evict hive cache");
        getAllTable();
        LOGGER.info("After evict hive cache, " +
                "automatically refresh hive tables cache.");
    }

    /**
     * Query Hive for Show tables or show databases, which will return List of String
     *
     * @param sql sql string
     * @return
     */
    private Iterable<String> queryHiveString(String sql) {
        List<String> res = new ArrayList<>();
        Statement stmt = null;
        ResultSet rs = null;

        try {
            Class.forName(hiveClassName);
            if (conn == null) {
                conn = DriverManager.getConnection(hiveUrl);
            }
            LOGGER.info("got connection");
            stmt = conn.createStatement();
            rs = stmt.executeQuery(sql);
            while (rs.next()) {
                res.add(rs.getString(1));
            }
        } catch (Exception e) {
            LOGGER.error("Query Hive JDBC has error, {}", e.getMessage());
        } finally {
            closeConnection(stmt, rs);
        }
        return res;
    }


    private void closeConnection(Statement stmt, ResultSet rs) {
        try {
            if (rs != null) {
                rs.close();
            }
            if (stmt != null) {
                stmt.close();
            }
            if (conn != null) {
                conn.close();
                conn = null;
            }
        } catch (SQLException e) {
            LOGGER.error("Close JDBC connection has problem. {}", e.getMessage());
        }
    }

    /**
     * Get the Hive table location from hive table metadata string
     *
     * @param tableMetadata hive table metadata string
     * @return Hive table location
     */
    public String getLocation(String tableMetadata) {
        tableMetadata = tableMetadata.toLowerCase();
        int index = tableMetadata.indexOf("location");
        if (index == -1) {
            return "";
        }

        int start = tableMetadata.indexOf("\'", index);
        int end = tableMetadata.indexOf("\'", start + 1);

        if (start == -1 || end == -1) {
            return "";
        }

        return tableMetadata.substring(start + 1, end);
    }

    /**
     * Get the Hive table schema: column name, column type, column comment
     * The input String looks like following:
     * <p>
     * CREATE TABLE `employee`(
     * `eid` int,
     * `name` string,
     * `salary` string,
     * `destination` string)
     * COMMENT 'Employee details'
     * ROW FORMAT SERDE
     * 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
     * WITH SERDEPROPERTIES (
     * 'field.delim'='\t',
     * 'line.delim'='\n',
     * 'serialization.format'='\t')
     * STORED AS INPUTFORMAT
     * 'org.apache.hadoop.mapred.TextInputFormat'
     * OUTPUTFORMAT
     * 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
     * LOCATION
     * 'file:/user/hive/warehouse/employee'
     * TBLPROPERTIES (
     * 'bucketing_version'='2',
     * 'transient_lastDdlTime'='1562086077')
     *
     * @param tableMetadata hive table metadata string
     * @return List of FieldSchema
     */
    public List<FieldSchema> getColums(String tableMetadata) {
        List<FieldSchema> res = new ArrayList<>();
        int start = tableMetadata.indexOf("(") + 1; // index of the first '('
        int end = tableMetadata.indexOf(")", start); // index of the first ')'
        String[] colsArr = tableMetadata.substring(start, end).split(",");
        for (String colStr : colsArr) {
            colStr = colStr.trim();
            String[] parts = colStr.split(" ");
            String colName = parts[0].trim().substring(1, parts[0].trim().length() - 1);
            String colType = parts[1].trim();
            String comment = getComment(colStr);
            FieldSchema schema = new FieldSchema(colName, colType, comment);
            res.add(schema);
        }
        return res;
    }

    /**
     * Parse one column string
     * <p>
     * Input example:
     * `merch_date` string COMMENT 'this is merch process date'
     *
     * @param colStr column string
     * @return
     */
    public String getComment(String colStr) {
        String pattern = "'([^\"|^\']|\"|\')*'";
        Matcher m = Pattern.compile(pattern).matcher(colStr.toLowerCase());
        if (m.find()) {
            String text = m.group();
            String result = text.substring(1, text.length() - 1);
            if (!result.isEmpty()) {
                LOGGER.info("Found value: " + result);
            }
            return result;
        } else {
            LOGGER.info("NO MATCH");
            return "";
        }
    }
}

```

### 1.2.2 vi ./${conf_dir}/HiveMetaStoreProxy.java
```
/*
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
*/

package org.apache.griffin.core.metastore.hive;

import javax.annotation.PreDestroy;

import org.apache.hadoop.hive.conf.HiveConf;
import org.apache.hadoop.hive.metastore.HiveMetaStoreClient;
import org.apache.hadoop.hive.metastore.IMetaStoreClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.security.UserGroupInformation;
import java.io.IOException;

@Component
public class HiveMetaStoreProxy {
    private static final Logger LOGGER = LoggerFactory
        .getLogger(HiveMetaStoreProxy.class);

    @Value("${hive.metastore.uris}")
    private String uris;

    /**
     * Set attempts and interval for HiveMetastoreClient to retry.
     *
     * @hive.hmshandler.retry.attempts: The number of times to retry a
     * HMSHandler call if there were a connection error
     * .
     * @hive.hmshandler.retry.interval: The time between HMSHandler retry
     * attempts on failure.
     */
    @Value("${hive.hmshandler.retry.attempts}")
    private int attempts;

    @Value("${hive.hmshandler.retry.interval}")
    private String interval;

    @Value("${hive.keytab.path}")
    private String keytabPath;

    @Value("${hive.krb5.path}")
    private String krb5Path;

    @Value("${hive.need.kerberos}")
    private String needKerberos;

    @Value("${hive.keytab.user}")
    private String keytabUser;

    private IMetaStoreClient client = null;

    // @PostConstruct
    public void init() {
        if (needKerberos != null && needKerberos.equalsIgnoreCase("true")) {
            LOGGER.info("Hive need Kerberos Auth.");

            System.setProperty("java.security.krb5.conf", krb5Path);
            Configuration conf = new Configuration();
            conf.set("hadoop.security.authentication", "Kerberos");
            UserGroupInformation.setConfiguration(conf);
            try {
                UserGroupInformation.loginUserFromKeytab(keytabUser, keytabPath);
            } catch (IOException e) {
                LOGGER.error("Register Kerberos has error. {}", e.getMessage());
            }
        }
    }

    @Bean
    public IMetaStoreClient initHiveMetastoreClient() {
        init();
        HiveConf hiveConf = new HiveConf();
        hiveConf.set("hive.metastore.local", "false");
        hiveConf.setIntVar(HiveConf.ConfVars.METASTORETHRIFTCONNECTIONRETRIES,
            3);
        hiveConf.setVar(HiveConf.ConfVars.METASTOREURIS, uris);
        hiveConf.setIntVar(HiveConf.ConfVars.HMSHANDLERATTEMPTS, attempts);
        hiveConf.setVar(HiveConf.ConfVars.HMSHANDLERINTERVAL, interval);
        try {
            client = HiveMetaStoreClient.newSynchronizedClient(new HiveMetaStoreClient(hiveConf));
        } catch (Exception e) {
            LOGGER.error("Failed to connect hive metastore. {}", e);
        }
        return client;
    }

    @PreDestroy
    public void destroy() {
        if (null != client) {
            client.close();
        }
    }
}

```

### 1.2.3 vi ./${conf_dir}/HiveMetastoreServiceJDBCImplTest.java # setUp()函数中添加 serviceJdbc.setHiveKrb5Path 行，
### 具体类似如下：
```
    public void setUp() throws SQLException {
        serviceJdbc.setConn(conn);
        serviceJdbc.setHiveClassName("org.apache.hive.jdbc.HiveDriver");
        serviceJdbc.setNeedKerberos("true");
        serviceJdbc.setKeytabPath("/path/to/keytab");
        serviceJdbc.setKeytabUser("user");
        serviceJdbc.setHiveKrb5Path("/etc/krb5.conf");
    }

```


### 1.2.4 vi griffin/measure/pom.xml # 先不更改 # 2.4.0+cdh6.3.2 # 无需操作该步骤
```
        <spark.major.version>2.4</spark.major.version>
        <spark.version>${spark.major.version}.0-cdh6.3.2</spark.version>
```

### 1.2.5 vi ./${conf_dir}/hibernate_mysql_pom.xml # 相关行修改后，类似如下
```
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <hadoop.version>3.0.0-cdh6.3.2</hadoop.version>
        <hive.version>2.1.1-cdh6.3.2</hive.version>
        <scala.version>2.11</scala.version>
        <spring.boot.version>1.5.15.RELEASE</spring.boot.version>
        <confluent.version>3.2.0</confluent.version>
        <quartz.version>2.2.3</quartz.version>
        <start-class>org.apache.griffin.core.GriffinWebApplication</start-class>
        <powermock.version>1.6.6</powermock.version>
        <mockito.version>1.10.19</mockito.version>
        <spring-boot-maven-plugin.version>1.5.15.RELEASE</spring-boot-maven-plugin.version>
        <derby.version>10.14.1.0</derby.version>
        <eclipselink.version>2.6.0</eclipselink.version>
```

### 1.2.6 vi ./${conf_dir}/pom.xml # 相关行修改后，类似如下
```
        <hadoop.version>3.0.0-cdh6.3.2</hadoop.version>
        <hive.version>2.1.1-cdh6.3.2</hive.version>
        <scala.version>2.11</scala.version>
        <spring.boot.version>2.1.7.RELEASE</spring.boot.version>
        <spring.security.kerberos.version>1.0.1.RELEASE</spring.security.kerberos.version>
        <confluent.version>3.2.0</confluent.version>
        <quartz.version>2.2.3</quartz.version>
        <start-class>org.apache.griffin.core.GriffinWebApplication</start-class>
        <powermock.version>2.0.2</powermock.version>
        <spring-boot-maven-plugin.version>2.1.7.RELEASE</spring-boot-maven-plugin.version>
        <derby.version>10.14.1.0</derby.version>
        <eclipselink.version>2.6.0</eclipselink.version>
        <mysql.java.version>5.1.47</mysql.java.version>
        <postgresql.version>9.4.1212.jre7</postgresql.version>
        <livy.core.version>0.3.0</livy.core.version>
        <elasticsearch-rest-client.version>6.2.4</elasticsearch-rest-client.version>
        <jackson-databind.version>2.9.9.3</jackson-databind.version>
```

### 1.2.7 vi ./${conf_dir}/pom.xml # Due to integration with CDH 6.3.2, inrepositoriesThe following sections are added:
### 注意：添加到 <repositories></repositories> 下
```
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
```

### 1.2.8 vi ./${conf_dir}/pom.xml ( 在 hive-jdbc 里排除 org.eclipse.jetty 的 jetty-runner ( 否则有包冲突 ) )
### 参考：https://www.5axxw.com/questions/content/b0z1cq
###      https://stackoverflow.com/questions/65721963/spring-boot-tomcat-jetty-application-failed-to-start
```
        <!-- to access Hive using JDBC -->
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-jdbc</artifactId>
            <version>${hive.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.eclipse.jetty.aggregate</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.eclipse.jetty.orbit</groupId>
                    <artifactId>javax.servlet</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>javax.servlet</groupId>
                    <artifactId>servlet-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.mortbay.jetty</groupId>
                    <artifactId>servlet-api-2.5</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.eclipse.jetty</groupId>
                    <artifactId>jetty-runner</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

```

### 1.2.9 vi ./${conf_dir}/application.properties
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
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#
spring.datasource.url=jdbc:mysql://onedts-dev-cdh-cm-v01.smartparkos.com:3306/quartz?autoReconnect=true&useSSL=false
spring.datasource.username=griffin
spring.datasource.password=Graffin@23456
spring.jpa.generate-ddl=true
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.jpa.show-sql=true
spring.jpa.open-in-view=false
spring.cache.cache-names=jdbcHive,hive
# Hive metastore
hive.metastore.uris=thrift://onedts-dev-cdh-master-v01.smartparkos.com:9083,thrift://onedts-dev-cdh-master-v02.smartparkos.com:9083
hive.metastore.dbname=bigdata
hive.hmshandler.retry.attempts=15
hive.hmshandler.retry.interval=2000ms
#Hive jdbc
hive.jdbc.className=org.apache.hive.jdbc.HiveDriver
# hive.jdbc.url=jdbc:hive2://onedts-dev-cdh-client-v01.smartparkos.com:25005/bigdata;principal=hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
hive.jdbc.url=jdbc:hive2://onedts-dev-cdh-client-v01.smartparkos.com:25005/
hive.need.kerberos=true
hive.keytab.user=hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM
hive.keytab.path=/home/griffin/security/keytabs/hive.keytab
hive.krb5.path=/etc/krb5.conf
# Hive cache time
cache.evict.hive.fixedRate.in.milliseconds=900000
# Kafka schema registry
kafka.schema.registry.url=http://onedts-dev-cdh-cm-v01.smartparkos.com:8081
# Update job instance state at regular intervals
jobInstance.fixedDelay.in.milliseconds=60000
# Expired time of job instance which is 7 days that is 604800000 milliseconds.Time unit only supports milliseconds
jobInstance.expired.milliseconds=604800000
# schedule predicate job every 5 minutes and repeat 12 times at most
#interval time unit s:second m:minute h:hour d:day,only support these four units
predicate.job.interval=5m
predicate.job.repeat.count=12
# external properties directory location
external.config.location=
# external BATCH or STREAMING env
external.env.location=
# login strategy ("default" or "ldap")
login.strategy=default
# ldap
ldap.url=ldap://onedts-dev-cdh-ipa-v01.smartparkos.com:389
ldap.email=@smartparkos.com
ldap.searchBase=cn=users,cn=accounts,dc=smartparkos,dc=com
ldap.searchPattern=(uid={0})
# hdfs default name
fs.defaultFS=hdfs://odtscluster
# elasticsearch
elasticsearch.host=onedts-dev-cdh-cm-v01.smartparkos.com
elasticsearch.port=19200
elasticsearch.scheme=http
# elasticsearch.user = user
# elasticsearch.password = password
# livy
livy.uri=http://onedts-dev-cdh-master-v01.smartparkos.com:8998/batches
livy.need.queue=false
livy.task.max.concurrent.count=20
livy.task.submit.interval.second=3
livy.task.appId.retry.count=3
livy.need.kerberos=true
livy.server.auth.kerberos.principal=livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
livy.server.auth.kerberos.keytab=/home/griffin/security/keytabs/livy.keytab
# yarn url
yarn.uri=http://onedts-dev-cdh-master-v01.smartparkos.com:8088
# griffin event listener
internal.event.listeners=GriffinJobEventHook
logging.file=logs/griffin-service.log
server.compression.enabled=true
server.compression.mime-types=application/json,application/xml,text/html,text/xml,text/plain,application/javascript,text/css

```

### 1.2.10 vi ./${conf_dir}/quartz.properties # 主要修改如下参数（其他参数不变)
```
org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate

```

### 1.2.11 vi ./${conf_dir}/sparkProperties.json
```
{
  "file": "hdfs:///griffin/jars/griffin-measure.jar",
  "className": "org.apache.griffin.measure.Application",
  "queue": "default",
  "numExecutors": 2,
  "executorCores": 1,
  "driverMemory": "1g",
  "executorMemory": "1g",
  "conf": {
    "spark.yarn.dist.files": "hdfs:///user/spark/spark_conf/hive-site.xml",
    "spark.driver.extraClassPath": "/opt/cloudera/parcels/CDH/lib/hive/lib/hive-exec.jar"
  },
  "files": [],
  "jars": []
}

```

### 1.2.12 vi ./${conf_dir}/env_batch.json
```
{
  "spark": {
    "log.level": "WARN",
    "checkpoint.dir" : "hdfs:///griffin/checkpoint/${JOB_NAME}"
  },
  "sinks": [
    {
      "name": "console",
      "type": "CONSOLE",
      "config": {
        "max.log.lines": 10
      }
    },
    {
      "name": "hdfs",
      "type": "HDFS",
      "config": {
        "path": "hdfs:///griffin/persist",
        "max.persist.lines": 10000,
        "max.lines.per.file": 10000
      }
    },
    {
      "name": "elasticsearch",
      "type": "ELASTICSEARCH",
      "config": {
        "method": "post",
        "api": "http://onedts-dev-cdh-cm-v01.smartparkos.com:19200/griffin/accuracy",
        "connection.timeout": "1m",
        "retry": 10
      }
    }
  ],
 "griffin.checkpoint": [
    {
      "type": "zk",
      "config": {
        "hosts": "onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181",
        "namespace": "griffin/infocache",
        "lock.path": "lock",
        "mode": "persist",
        "init.clear": true,
        "close.clear": false
      }
    }
  ]
}

```

### 1.2.13 vi ./${conf_dir}/env_streaming.json
```
{
  "spark": {
    "log.level": "WARN",
    "checkpoint.dir": "hdfs:///griffin/checkpoint/${JOB_NAME}",
    "init.clear": true,
    "batch.interval": "1m",
    "process.interval": "5m",
    "config": {
      "spark.default.parallelism": 4,
      "spark.task.maxFailures": 5,
      "spark.streaming.kafkaMaxRatePerPartition": 1000,
      "spark.streaming.concurrentJobs": 4,
      "spark.yarn.maxAppAttempts": 5,
      "spark.yarn.am.attemptFailuresValidityInterval": "1h",
      "spark.yarn.max.executor.failures": 120,
      "spark.yarn.executor.failuresValidityInterval": "1h",
      "spark.hadoop.fs.hdfs.impl.disable.cache": true
    }
  },
  "sinks": [
    {
      "type": "CONSOLE",
      "config": {
        "max.log.lines": 100
      }
    },
    {
      "type": "HDFS",
      "config": {
        "path": "hdfs:///griffin/persist",
        "max.persist.lines": 10000,
        "max.lines.per.file": 10000
      }
    },
    {
      "type": "ELASTICSEARCH",
      "config": {
        "method": "post",
        "api": "http://onedts-dev-cdh-cm-v01.smartparkos.com:19200/griffin/accuracy"
      }
    }
  ],
  "griffin.checkpoint": [
    {
      "type": "zk",
      "config": {
        "hosts": "onedts-dev-cdh-master-v01.smartparkos.com:2181,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181",
        "namespace": "griffin/infocache",
        "lock.path": "lock",
        "mode": "persist",
        "init.clear": true,
        "close.clear": false
      }
    }
  ]
}

```

### 1.2.14 vi ./${conf_dir}/dq.json
```
{
  "name": "streaming_accu",
  "process.type": "streaming",
  "data.sources": [
    {
      "name": "src",
      "baseline": true,
      "connectors": [
        {
          "type": "kafka",
          "version": "0.8",
          "config": {
            "kafka.config": {
              "bootstrap.servers": "onedts-dev-cdh-datanode-v01.smartparkos.com:9092,onedts-dev-cdh-datanode-v02.smartparkos.com:9092,onedts-dev-cdh-datanode-v03.smartparkos.com:9092",
              "group.id": "griffin",
              "auto.offset.reset": "largest",
              "auto.commit.enable": "false"
            },
            "topics": "source",
            "key.type": "java.lang.String",
            "value.type": "java.lang.String"
          },
          "pre.proc": [
            {
              "dsl.type": "df-opr",
              "rule": "from_json"
            }
          ]
        }
      ],
      "checkpoint": {
        "type": "json",
        "file.path": "hdfs:///griffin/streaming/dump/source",
        "info.path": "source",
        "ready.time.interval": "10s",
        "ready.time.delay": "0",
        "time.range": ["-5m", "0"],
        "updatable": true
      }
    }, {
      "name": "tgt",
      "connectors": [
        {
          "type": "kafka",
          "version": "0.8",
          "config": {
            "kafka.config": {
              "bootstrap.servers": "onedts-dev-cdh-datanode-v01.smartparkos.com:9092,onedts-dev-cdh-datanode-v02.smartparkos.com:9092,onedts-dev-cdh-datanode-v03.smartparkos.com:9092",
              "group.id": "griffin",
              "auto.offset.reset": "largest",
              "auto.commit.enable": "false"
            },
            "topics": "target",
            "key.type": "java.lang.String",
            "value.type": "java.lang.String"
          },
          "pre.proc": [
            {
              "dsl.type": "df-opr",
              "rule": "from_json"
            }
          ]
        }
      ],
      "checkpoint": {
        "type": "json",
        "file.path": "hdfs:///griffin/streaming/dump/target",
        "info.path": "target",
        "ready.time.interval": "10s",
        "ready.time.delay": "0",
        "time.range": ["-1m", "0"]
      }
    }
  ],
  "evaluate.rule": {
    "rules": [
      {
        "dsl.type": "griffin-dsl",
        "dq.type": "ACCURACY",
        "out.dataframe.name": "accu",
        "rule": "src.id = tgt.id AND src.name = tgt.name AND src.color = tgt.color AND src.time = tgt.time",
        "details": {
          "source": "src",
          "target": "tgt",
          "miss": "miss_count",
          "total": "total_count",
          "matched": "matched_count"
        },
        "out":[
          {
            "type":"metric",
            "name": "accu"
          },
          {
            "type":"record",
            "name": "missRecords"
          }
        ]
      }
    ]
  },
  "sinks": ["CONSOLE", "HDFS", "ELASTICSEARCH"]
}

```

### 1.2.15 编译 # 注意：建议用普通用户编译，不用 root 用户编译
```
chown -R griffin:griffin /DATA/disk1/software/griffin;
su - griffin

conf_dir=conf_bak_060;
cd /DATA/disk1/software/griffin/griffin-0.6.0/;
\cp ../${conf_dir}/application.properties ./service/src/main/resources/;
# \cp ../${conf_dir}/application.properties ./service/src/test/resources/;
\cp ../${conf_dir}/quartz.properties ./service/src/main/resources/;

\cp ../${conf_dir}/sparkProperties.json ./service/src/main/resources/;
\cp ../${conf_dir}/env_batch.json ./service/src/main/resources/env/;
\cp ../${conf_dir}/env_streaming.json ./service/src/main/resources/env/;

\cp ../${conf_dir}/logo.png ./ui/angular/src/assets/img/;
\cp ../${conf_dir}/index.html ./ui/angular/src/;
\cp ../${conf_dir}/test.html ./ui/angular/src/;

\cp ../${conf_dir}/pom.xml ./service/;
\cp ../${conf_dir}/hibernate_mysql_pom.xml ./service/;
\cp ../${conf_dir}/HiveMetaStoreServiceJdbcImpl.java ./service/src/main/java/org/apache/griffin/core/metastore/hive/;
\cp ../${conf_dir}/HiveMetaStoreProxy.java ./service/src/main/java/org/apache/griffin/core/metastore/hive/;
\cp ../${conf_dir}/HiveMetastoreServiceJDBCImplTest.java ./service/src/test/java/org/apache/griffin/core/metastore/hive/;

mvn clean install

# 如果报错，则添加 -DskipTests
mvn clean install -DskipTests

# 如果需要查看详细调试信息，带 -X 参数，类似如下：
mvn -X clean install -DskipTests

```

# -------------------------------------------------------------------
## 1.3 创建 griffin 用户 及生成票据文件 # 在 onedts-dev-cdh-client-v01 服务器执行
```
kinit admin

ipa_user=griffin
ipa user-add ${ipa_user} --first=${ipa_user} --last=${ipa_user} --shell=/bin/bash --homedir=/home/${ipa_user}

# 生成票据文件，在 onedts-dev-cdh-client-v01 服务器操作
cd /DATA/disk1/home/;
cp -r atlas griffin
mkdir -p ./griffin/security/keytabs/;
chown -R griffin:griffin ./griffin;

cd /home/;
ln -s /DATA/disk1/home/griffin ./griffin;

su - griffin
cd ~/security/keytabs/;
user_name=griffin
ipa_server=onedts-dev-cdh-ipa-v01.smartparkos.com
user_home=/DATA/disk1/home/${user_name};
ipa-getkeytab -s ${ipa_server} \
-p  ${user_name} \
-e aes256-cts-hmac-sha1-96,aes128-cts-hmac-sha1-96,aes256-cts-hmac-sha384-192,aes128-cts-hmac-sha256-128,des3-cbc-sha1,arcfour-hmac \
-k ${user_home}/security/keytabs/${user_name}.keytab

```

# -------------------------------------------------------------------
## 1.4 解压 安装 griffin
```
mkdir /DATA/disk1/apps/;
cd /DATA/disk1/software/griffin/griffin-0.6.0/;
rm -rf /DATA/disk1/apps/service-0.6.0;
tar -xvf ./target/service-0.6.0.tar.gz -C /DATA/disk1/apps/;
chown -R griffin:griffin /DATA/disk1/apps/service-0.6.0;
chown -R griffin:griffin /DATA/disk1/software/griffin/griffin-0.6.0;

cd /DATA/disk1/apps/;
unlink griffin;
ln -s ./service-0.6.0 ./griffin;

```

### 1.4.1 先备份 config 目录
```
su - griffin;

cd /DATA/disk1/apps/griffin/;
cdate=$(date '+%Y%m%d')
\cp -r config config.bak.${cdate};

```

### 1.4.2 hive-site.xml
```
su - griffin;

cd /DATA/disk1/apps/griffin/config/;
\mv hive-site.xml hive-site.xml.bak;
ln -s /etc/hive/conf/hive-site.xml ./hive-site.xml;

```

### 1.4.3 创建 quartz 数据库 及 griffin 用户
```
drop database if exists quartz;
CREATE DATABASE `quartz` /*!40100 DEFAULT CHARACTER SET latin1 */;
grant all privileges on quartz.* to griffin@'%' identified by "Graffin@23456";
flush privileges;

mysql -honedts-dev-cdh-cm-v01.smartparkos.com -ugriffin -pGraffin\@23456
use quartz;
source /DATA/disk1/software/griffin/griffin-0.6.0/service/target/classes/Init_quartz_mysql_innodb.sql;

```

### 1.4.4 kafka 创建 topic 及 授权
```
su - hadoop
kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181 \
--create --replication-factor 2 --partitions 1 \
--topic source

kafka-topics \
--zookeeper onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com:2181,onedts-dev-cdh-client-v01.smartparkos.com:2181 \
--create --replication-factor 2 --partitions 1 \
--topic target

kafka-sentry -cr -r griffin
# 给 onedts-dev-cdh-client-v01 服务器(ip: 192.168.100.105) 授权
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=source->action=read"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=source->action=write"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=source->action=describe"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=target->action=read"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=target->action=write"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Topic=target->action=describe"

kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Consumergroup=griffin->action=describe"
kafka-sentry -gpr -r griffin -p "Host=192.168.100.105->Consumergroup=griffin->action=read"

kafka-sentry -arg -r griffin -g griffin

```

### 1.4.5 相关依赖操作
```
# *1). 创建相关 hdfs 目录
su - hadoop;
hdfs dfs -rmr /griffin/*;
hdfs dfs -mkdir -p /griffin/jars/;
hdfs dfs -mkdir -p /griffin/persist/;
hdfs dfs -mkdir -p /griffin/checkpoint/;
hdfs dfs -mkdir -p /griffin/stream/;
hdfs dfs -chown -R griffin:griffin /griffin/;

hdfs dfs -mkdir -p /user/spark/spark_conf/;
hdfs dfs -chown -R griffin:griffin /user/spark/spark_conf/;

# *2). 上传 griffin-measure.jar 到 hdfs
su - griffin;
cd /DATA/disk1/software/griffin/griffin-0.6.0/; 
\cp ./measure/target/measure-0.6.0.jar  griffin-measure.jar; 
hdfs dfs -put ./griffin-measure.jar /griffin/jars/;
# 注意：以上操作需要编译完成后执行 ( 编译成功后，将生成 measure/target/measure-0.6.0.jar 文件 )

# *3). 上传 env_streaming.json & dq.json 配置文件到 hdfs 的 /griffin/stream/ 目录
# 注意： dq.json 文件内容，参考 1.4.4 步
cd /DATA/disk1/apps/griffin/config/;
\cp /DATA/disk1/software/griffin/conf_bak_060/dq.json ./;

hdfs dfs -put /DATA/disk1/apps/griffin/config/env/env_streaming.json /griffin/stream/;
hdfs dfs -put /DATA/disk1/apps/griffin/config/dq.json /griffin/stream/;

# *4). 将 hive-site.xml 文件 put 到 hdfs, 已操作
hdfs dfs -put /etc/hive/conf/hive-site.xml /user/spark/spark_conf/;

# *5). spark jars
cd /opt/cloudera/parcels/CDH/lib/spark/jars/;
hdfs dfs -rmr /user/spark/jars/*;
hdfs dfs -put ./*.jar /user/spark/jars/;

# *6). 启动 ES 到 19200 端口

```

### 1.4.6 创建相关ES索引 (注意：ES 安装略)
###       You need to create Elasticsearch index in advance, in order to set number of shards, replicas, and other settings to desired values:
```
curl -k -H "Content-Type: application/json" -X PUT http://onedts-dev-hdp-datanode-v01.smartparkos.com:19200/griffin \
 -d '{
    "aliases": {},
    "mappings": {
        "accuracy": {
            "properties": {
                "name": {
                    "fields": {
                        "keyword": {
                            "ignore_above": 256,
                            "type": "keyword"
                        }
                    },
                    "type": "text"
                },
                "tmst": {
                    "type": "date"
                }
            }
        }
    },
    "settings": {
        "index": {
            "number_of_replicas": "2",
            "number_of_shards": "5"
        }
    }
}'

```

### 1.4.7 add zk node
```
cd /opt/cloudera/parcels/CDH/lib/zookeeper/bin/;
./zkCli.sh

create /griffin griffin
create /griffin/infocache infocache

getAcl /griffin
getAcl /griffin/infocache

```

### 1.4.8 添加环境变量到 griffin 用户的 ~/.bash_profile
###       vi ~/.bash_profile
```
#!/bin/bash

export HADOOP_CONF_DIR=/etc/hadoop/conf
export HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export HADOOP_CLASSPATH=`${HADOOP_HOME}/bin/hadoop classpath`
export HADOOP_OPTS="-Djava.net.preferIPv4Stack=true -Djava.library.path=/opt/cloudera/parcels/CDH/lib/hadoop/lib/native $HADOOP_OPTS"
export SPARK_HOME=/opt/cloudera/parcels/CDH/lib/spark

# export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
# export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
export HADOOP_HOME=/opt/cloudera/parcels/CDH/lib/hadoop
export HADOOP_COMMON_HOME=${HADOOP_HOME}
export HADOOP_COMMON_LIB_NATIVE_DIR=${HADOOP_HOME}/lib/native
export HADOOP_HDFS_HOME=${HADOOP_HOME}
export HADOOP_INSTALL=${HADOOP_HOME}
export HADOOP_MAPRED_HOME=${HADOOP_HOME}
export HADOOP_USER_CLASSPATH_FIRST=true
export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop
export YARN_CONF_DIR=${HADOOP_HOME}/etc/hadoop
export SPARK_HOME=/opt/cloudera/parcels/CDH/lib/spark
export LIVY_HOME=/opt/cloudera/parcels/LIVY
export HIVE_HOME=/opt/cloudera/parcels/CDH/lib/hive
export YARN_HOME=${HADOOP_HOME}
export SPARK_CONF_DIR=$SPARK_HOME/conf
export HBASE_HOME=/opt/cloudera/parcels/CDH/lib/hbase
export SCALA_HOME=/usr/local/scala
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HBASE_HOME/bin:$SCALA_HOME/bin:$PATH
export CLASSPATH=$CLASSPATH:$HADOOP_HOME/lib/*:.

```

### 1.4.9 启动 griffin
```
killall -u griffin;
su - griffin;

# start service # 启动前记得将 mysql-jdbc 驱动 copy 到 ./lib 目录下
source ~/.bash_profile;
cd /DATA/disk1/apps/griffin/lib/;
\cp /usr/share/java/mysql-connector-java-5.1.47.jar ./;

conf_dir=conf_bak_060;
cd /DATA/disk1/apps/griffin/config/;
\cp /DATA/disk1/software/griffin/${conf_dir}/dq.json ./

jps|grep GriffinWebApplication|awk '{print $1}'|xargs kill -9
cd /DATA/disk1/apps/griffin/;
rm -rf ./logs/*;

./bin/griffin.sh start
```

# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 二. 测试 用 bigdata 用户测试(其他用户的话，需要有相关权限)
```

drop table if exists griffin_test_src;
use bigdata;
create table griffin_test_src(id int, cname varchar(20), age int) stored as parquet;
insert into griffin_test_src(id,cname,age)
values(1,'luoyoumou',42);
insert into griffin_test_src(id,cname,age)
values(2,'zhongchangtian',42);
insert into griffin_test_src(id,cname,age)
values(3,'hanqiguang',38);
insert into griffin_test_src(id,cname,age)
values(4,'tangyuguang',38);
insert into griffin_test_src(id,cname,age)
values(5,'zengxingxing',36);

create table griffin_test_tgt(id int, cname varchar(20), age int) stored as parquet;
insert into griffin_test_tgt(id,cname,age)
values(1,'luoyoumou',42);
insert into griffin_test_tgt(id,cname,age)
values(2,'zhongchangtian',42);
insert into griffin_test_tgt(id,cname,age)
values(3,'hanqiguang',38);
insert into griffin_test_tgt(id,cname,age)
values(4,'tangyuguang',38);

drop table if exists griffin_test_pt_src;
drop table if exists griffin_test_pt_tgt;

create table griffin_test_pt_src(id int, cname varchar(20), age int) 
PARTITIONED BY(dt int) 
stored as parquet;
INSERT OVERWRITE TABLE griffin_test_pt_src PARTITION(dt=20210608)
select id,cname,age 
from griffin_test_src;
INSERT OVERWRITE TABLE griffin_test_pt_src PARTITION(dt=20210609)
select id,cname,age 
from griffin_test_src;

create table griffin_test_pt_tgt(id int, cname varchar(20), age int) 
PARTITIONED BY(dt int) 
stored as parquet;
INSERT OVERWRITE TABLE griffin_test_pt_tgt PARTITION(dt=20210608)
select id,cname,age 
from griffin_test_tgt;
INSERT OVERWRITE TABLE griffin_test_pt_tgt PARTITION(dt=20210609)
select id,cname,age 
from griffin_test_tgt;
```


# -------------------------------------------------------------------
# -------------------------------------------------------------------
# 三. 错误解决

# -------------------------------------------------------------------
## 3.1.1 报错如下
```
2021-06-04 17:04:24.923  INFO 30717 --- [           main] o.a.g.c.m.h.HiveMetaStoreService        [106]  : Hive need Kerberos Auth.
2021-06-04 17:04:24.976  INFO 30717 --- [           main] o.a.h.s.UserGroupInformation            [1147]  : Login successful for user hive/onedts-dev-cdh-client-v01.smartparkos.com@SMARTPARKOS.COM using keytab file /home/griffin/security/keytabs/hive.keytab. Keytab auto renewal enabled : false
2021-06-04 17:04:24.996  INFO 30717 --- [           main] o.a.h.h.c.HiveConf                      [188]  : Found configuration file file:/DATA/disk1/apps/service-0.6.0/config/hive-site.xml
2021-06-04 17:04:25.119  INFO 30717 --- [           main] h.metastore                             [308]  : HMS client filtering is enabled.
2021-06-04 17:04:25.120  INFO 30717 --- [           main] h.metastore                             [472]  : Trying to connect to metastore with URI thrift://onedts-dev-cdh-master-v01.smartparkos.com:9083
2021-06-04 17:04:25.173  INFO 30717 --- [           main] h.metastore                             [546]  : Opened a connection to metastore, current connections: 1
2021-06-04 17:04:25.174  INFO 30717 --- [           main] h.metastore                             [599]  : Connected to metastore.
2021-06-04 17:04:25.249  WARN 30717 --- [           main] aWebConfiguration$JpaWebMvcConfiguration[226]  : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2021-06-04 17:04:25.495  INFO 30717 --- [           main] o.s.s.c.ThreadPoolTaskExecutor          [171]  : Initializing ExecutorService 'applicationTaskExecutor'
2021-06-04 17:04:25.583  INFO 30717 --- [           main] o.s.b.a.w.s.WelcomePageHandlerMapping   [54]  : Adding welcome page: class path resource [public/index.html]
2021-06-04 17:04:25.755  INFO 30717 --- [           main] o.s.s.c.ThreadPoolTaskScheduler         [171]  : Initializing ExecutorService 'taskScheduler'
2021-06-04 17:04:25.786  INFO 30717 --- [           main] o.s.s.q.SchedulerFactoryBean            [727]  : Starting Quartz Scheduler now
2021-06-04 17:04:25.804  INFO 30717 --- [           main] o.q.c.QuartzScheduler                   [575]  : Scheduler spring-boot-quartz_$_onedts-dev-cdh-client-v011622797464230 started.
2021-06-04 17:04:25.816 ERROR 30717 --- [   scheduling-1] o.s.s.s.TaskUtils$LoggingErrorHandler   [96]  : Unexpected error occurred in scheduled task.

java.lang.IllegalArgumentException: Cannot find cache named 'jdbcHive' for Builder[public void org.apache.griffin.core.metastore.hive.HiveMetaStoreServiceJdbcImpl.evictHiveCache()] caches=[jdbcHive] | key='' | keyGenerator='cacheKeyGenerator' | cacheManager='' | cacheResolver='' | condition='',true,true
	at org.springframework.cache.interceptor.AbstractCacheResolver.resolveCaches(AbstractCacheResolver.java:92) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.getCaches(CacheAspectSupport.java:252) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport$CacheOperationContext.<init>(CacheAspectSupport.java:707) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.getOperationContext(CacheAspectSupport.java:265) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport$CacheOperationContexts.<init>(CacheAspectSupport.java:598) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:345) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheInterceptor.invoke(CacheInterceptor.java:61) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) ~[spring-aop-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:689) ~[spring-aop-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.apache.griffin.core.metastore.hive.HiveMetaStoreServiceJdbcImpl$$EnhancerBySpringCGLIB$$85ba9566.evictHiveCache(<generated>) ~[service-0.6.0.jar:0.6.0]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_181]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_181]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_181]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_181]
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:84) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54) [spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_181]
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308) [?:1.8.0_181]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [?:1.8.0_181]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [?:1.8.0_181]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_181]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_181]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_181]

2021-06-04 17:04:25.822 ERROR 30717 --- [   scheduling-1] o.s.s.s.TaskUtils$LoggingErrorHandler   [96]  : Unexpected error occurred in scheduled task.

java.lang.IllegalArgumentException: Cannot find cache named 'hive' for Builder[public void org.apache.griffin.core.metastore.hive.HiveMetaStoreServiceImpl.evictHiveCache()] caches=[hive] | key='' | keyGenerator='cacheKeyGenerator' | cacheManager='' | cacheResolver='' | condition='',true,true
	at org.springframework.cache.interceptor.AbstractCacheResolver.resolveCaches(AbstractCacheResolver.java:92) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.getCaches(CacheAspectSupport.java:252) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport$CacheOperationContext.<init>(CacheAspectSupport.java:707) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.getOperationContext(CacheAspectSupport.java:265) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport$CacheOperationContexts.<init>(CacheAspectSupport.java:598) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheAspectSupport.execute(CacheAspectSupport.java:345) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.cache.interceptor.CacheInterceptor.invoke(CacheInterceptor.java:61) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) ~[spring-aop-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:689) ~[spring-aop-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.apache.griffin.core.metastore.hive.HiveMetaStoreServiceImpl$$EnhancerBySpringCGLIB$$fd7c7a35.evictHiveCache(<generated>) ~[service-0.6.0.jar:0.6.0]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_181]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_181]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_181]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_181]
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:84) ~[spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54) [spring-context-5.1.10.RELEASE.jar:5.1.10.RELEASE]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [?:1.8.0_181]
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308) [?:1.8.0_181]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [?:1.8.0_181]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [?:1.8.0_181]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_181]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_181]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_181]

2021-06-04 17:04:25.847  INFO 30717 --- [           main] o.s.b.w.e.t.TomcatWebServer             [202]  : Tomcat started on port(s): 8080 (http) with context path ''
2021-06-04 17:04:25.849  INFO 30717 --- [           main] o.a.g.c.GriffinWebApplication           [59]  : Started GriffinWebApplication in 5.759 seconds (JVM running for 6.303)
[EL Fine]: sql: 2021-06-04 17:04:25.851--ServerSession(254145674)--Connection(1677128000)--SELECT ID, APPID, APPURI, CREATEDDATE, DELETED, expire_timestamp, MODIFIEDDATE, predicate_job_deleted, predicate_group_name, predicate_job_name, SESSIONID, STATE, timestamp, TRIGGERKEY, TYPE, job_id FROM JOBINSTANCEBEAN WHERE (expire_timestamp <= ?)
	bind => [1622797465822]
2021-06-04 17:04:25.852  INFO 30717 --- [           main] o.a.g.c.GriffinWebApplication           [37]  : application started
[EL Fine]: sql: 2021-06-04 17:04:25.869--ClientSession(360082003)--Connection(1061220936)--DELETE FROM JOBINSTANCEBEAN WHERE ((expire_timestamp <= ?) AND (DELETED = ?))
	bind => [1622797465822, false]
2021-06-04 17:04:25.876  INFO 30717 --- [   scheduling-1] o.a.g.c.j.JobServiceImpl                [351]  : Delete 0 expired job instances.
[EL Fine]: sql: 2021-06-04 17:04:25.878--ServerSession(254145674)--Connection(686972593)--SELECT DISTINCT ID, APPID, APPURI, CREATEDDATE, DELETED, expire_timestamp, MODIFIEDDATE, predicate_job_deleted, predicate_group_name, predicate_job_name, SESSIONID, STATE, timestamp, TRIGGERKEY, TYPE, job_id FROM JOBINSTANCEBEAN WHERE (STATE IN (?,?,?,?,?,?))
	bind => [STARTING, NOT_STARTED, RECOVERING, IDLE, RUNNING, BUSY]
2021-06-04 17:04:45.804  INFO 30717 --- [_ClusterManager] o.s.s.q.LocalDataSourceJobStore         [3579]  : ClusterManager: detected 1 failed or restarted instances.
2021-06-04 17:04:45.805  INFO 30717 --- [_ClusterManager] o.s.s.q.LocalDataSourceJobStore         [3438]  : ClusterManager: Scanning for instance "onedts-dev-cdh-client-v011622796399827"'s failed in-progress jobs.
[EL Fine]: sql: 2021-06-04 17:05:25.88--ServerSession(254145674)--Connection(1792347246)--SELECT DISTINCT ID, APPID, APPURI, CREATEDDATE, DELETED, expire_timestamp, MODIFIEDDATE, predicate_job_deleted, predicate_group_name, predicate_job_name, SESSIONID, STATE, timestamp, TRIGGERKEY, TYPE, job_id FROM JOBINSTANCEBEAN WHERE (STATE IN (?,?,?,?,?,?))
	bind => [STARTING, NOT_STARTED, RECOVERING, IDLE, RUNNING, BUSY]

```

## 3.1.2 Griffin 开启调试模式 vi ./bin/griffin.sh
```
JAVA_OPTS="-Dlogging.level.org.springframework.web=DEBUG -Dlogging.level.org.hibernate=DEBUG"
```

## 3.1.3 解决：vi config/application.properties # 添加如下参数 # 参考：https://blog.csdn.net/qq_42168543/article/details/110667683
```
spring.cache.cache-names=jdbcHive,hive
```

# -------------------------------------------------------------------
## 3.2.1 报错如下
```
ome/griffin/security/keytabs/livy.keytab
2021-06-08 18:15:13.793  INFO 15558 --- [http-nio-8080-exec-1] o.a.g.c.j.LivyTaskSubmitHelper          [308] : {"id":0,"name":null,"owner":null,"proxyUser":null,"state":"dead","appId":"application_1622515365210_0423","appInfo":{"driverLogUrl":null,"sparkUiUrl":"http://onedts-dev-cdh-master-v01.smartparkos.com:8088/cluster/app/application_1622515365210_0423"},"log":["\nYARN Diagnostics: ","Application application_1622515365210_0423 failed 2 times in previous 3600000 milliseconds due to AM Container for appattempt_1622515365210_0423_000002 exited with  exitCode: -1000","Failing this attempt.Diagnostics: [2021-06-08 18:15:09.217]Application application_1622515365210_0423 initialization failed (exitCode=255) with output: main : command provided 0","main : run as user is livy","main : requested yarn user is livy","Can't create directory /DATA/disk1/cdh-data/yarn/nm/usercache/livy/appcache/application_1622515365210_0423 - Permission denied","Did not create any app directories","","For more detailed output, check the application tracking page: http://onedts-dev-cdh-master-v01.smartparkos.com:8088/cluster/app/application_1622515365210_0423 Then click on links to logs of each attempt.",". Failing the application."]}
2021-06-08 18:15:13.793  INFO 15558 --- [http-nio-8080-exec-1] o.a.g.c.j.JobServiceImpl                [541] : {"id":0,"name":null,"owner":null,"proxyUser":null,"state":"dead","appId":"application_1622515365210_0423","appInfo":{"driverLogUrl":null,"sparkUiUrl":"http://onedts-dev-cdh-master-v01.smartparkos.com:8088/cluster/app/application_1622515365210_0423"},"log":["\nYARN Diagnostics: ","Application application_1622515365210_0423 failed 2 times in previous 3600000 milliseconds due to AM Container for appattempt_1622515365210_0423_000002 exited with  exitCode: -1000","Failing this attempt.Diagnostics: [2021-06-08 18:15:09.217]Application application_1622515365210_0423 initialization failed (exitCode=255) with output: main : command provided 0","main : run as user is livy","main : requested yarn user is livy","Can't create directory /DATA/disk1/cdh-data/yarn/nm/usercache/livy/appcache/application_1622515365210_0423 - Permission denied","Did not create any app directories","","For more detailed output, check the application tracking page: http://onedts-dev-cdh-master-v01.smartparkos.com:8088/cluster/app/application_1622515365210_0423 Then click on links to logs of each attempt.",". Failing the application."]}

```

## 3.2.2 解决：授权目录 /DATA/disk1/cdh-data/yarn/nm/usercache
```
sh ./ssh_to_all_node.sh "chmod 777 /DATA/disk1/cdh-data/yarn/nm/usercache;" datanode
```
# -------------------------------------------------------------------
## 3.3.1 报错如下
```
2021-06-09 09:35:15.488  INFO 10233 --- [nio-8080-exec-4] o.a.g.c.j.JobServiceImpl                [541]  : {"id":4,"name":null,"owner":null,"proxyUser":null,"state":"starting","appId":"application_1623201080231_0010","appInfo":{"driverLogUrl":null,"sparkUiUrl":null},"log":["21/06/09 09:35:14 INFO util.Utils: Using initial executors = 2, max of spark.dynamicAllocation.initialExecutors, spark.dynamicAllocation.minExecutors and spark.executor.instances","21/06/09 09:35:14 WARN cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: Attempted to request executors before the AM has registered!","21/06/09 09:35:14 WARN lineage.LineageWriter: Lineage directory /var/log/spark/lineage doesn't exist or is not writable. Lineage for this application will be disabled.","21/06/09 09:35:14 INFO util.Utils: Extension com.cloudera.spark.lineage.NavigatorAppListener not being initialized.","21/06/09 09:35:14 INFO logging.DriverLogger$DfsAsyncWriter: Started driver log file sync to: /user/spark/driverLogs/application_1623201080231_0010_driver.log","21/06/09 09:35:14 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all","21/06/09 09:35:14 INFO cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: ApplicationMaster registered as NettyRpcEndpointRef(spark-client://YarnAM)","21/06/09 09:35:15 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all","\nstderr: ","\nYARN Diagnostics: "]}
```

## 3.3.2 解决：
```
sh ./ssh_to_all_node.sh "mkdir -p /var/log/spark/;chown spark:spark /var/log/spark;" datanode
sh ./ssh_to_all_node.sh "mkdir -p /var/log/spark/lineage;chown spark:spark /var/log/spark/lineage;" datanode
```

# -------------------------------------------------------------------
## 3.4.1 无任何报错，但 griffin 提交任务一直是 DEAD 状态
```

```

## 3.4.2 解决
## "core-site.xml 的群集范围高级配置代码段（安全阀）HDFS（服务范围）"添加
```
hadoop.proxyuser.griffin.hosts=*
hadoop.proxyuser.griffin.groups=*
```

# -------------------------------------------------------------------
## 3.5.1 无任何报错，但 griffin 提交任务一直是 DEAD 状态, livy 报错类似如下（livy url: http://onedts-dev-cdh-master-v01.smartparkos.com:8998/ui/batch/2/log )
```
21/06/09 10:23:33 ERROR datasource.DataSource: load data source [source] fails
Exception in thread "main" org.apache.hadoop.security.AccessControlException: Permission denied: user=livy, access=READ_EXECUTE, inode="/user/hive/warehouse/bigdata.db/griffin_test_pt_src/dt=20210608":hive:hive:drwxrwx--x
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:400)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:262)
	at org.apache.sentry.hdfs.SentryINodeAttributesProvider$SentryPermissionEnforcer.checkPermission(SentryINodeAttributesProvider.java:86)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:194)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1855)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1839)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPathAccess(FSDirectory.java:1789)
	at org.apache.hadoop.hdfs.server.namenode.FSDirStatAndListingOp.getListingInt(FSDirStatAndListingOp.java:79)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getListing(FSNamesystem.java:3735)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.getListing(NameNodeRpcServer.java:1138)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.getListing(ClientNamenodeProtocolServerSideTranslatorPB.java:708)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:523)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:991)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:869)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:815)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1875)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2675)
```

## 3.5.2 解决: 参考：https://www.ibm.com/docs/en/watson-studio-local/1.2.1?topic=sucdhcwdl-protect-hive-data-access-it-securely-from-data-science-experience-local
```
hdfs dfs -setfacl -R -m user:livy:r-x /user/hive/warehouse/bigdata.db;
hdfs dfs -setfacl -R -m user:bob:r-x /user/hive/warehouse/bigdata.db/dsx_emp

# 由于配置了 Sentry 权限控制，上面操作无校，故，解决如下：
# 创建数据库相关角色
create role all_read;
grant select on database bigdata to role all_read;
grant select on database danalysis to role all_read;
grant select on database default to role all_read;
grant select on database dim_odts to role all_read;
grant select on database dim_odts_chengshi_jianshe to role all_read;
grant select on database dim_odts_daolu_jiaotong to role all_read;
grant select on database dim_odts_gonggong_anquan to role all_read;
grant select on database dim_odts_jiaoyu_keji to role all_read;
grant select on database dim_odts_jigou_tuanti to role all_read;
grant select on database dim_odts_jingji_jianshe to role all_read;
grant select on database dim_odts_mingsheng_fuwu to role all_read;
grant select on database dim_odts_shehui_fazhan to role all_read;
grant select on database dim_odts_shehui_ziuan to role all_read;
grant select on database dim_odts_weisheng_jiankang to role all_read;
grant select on database dim_odts_wenhua_xiuxian to role all_read;
grant select on database dim_odts_ziyuan_huanjing to role all_read;
grant select on database dim_yttg to role all_read;
grant select on database etl to role all_read;
grant select on database external_partitions to role all_read;
grant select on database linkedsee to role all_read;

grant role all_read to group griffin;
grant role all_read to group livy;

```

# -------------------------------------------------------------------
## 3.6.1 livy 报错类似如下（livy url: http://onedts-dev-cdh-master-v01.smartparkos.com:8998/ui/batch/2/log )
```
Caused by: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.AccessControlException): Permission denied: user=livy, access=WRITE, inode="/griffin/persist":griffin:griffin:drwxr-xr-x
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:400)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:256)
	at org.apache.sentry.hdfs.SentryINodeAttributesProvider$SentryPermissionEnforcer.checkPermission(SentryINodeAttributesProvider.java:86)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkPermission(FSPermissionChecker.java:194)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1855)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkPermission(FSDirectory.java:1839)
	at org.apache.hadoop.hdfs.server.namenode.FSDirectory.checkAncestorAccess(FSDirectory.java:1798)
	at org.apache.hadoop.hdfs.server.namenode.FSDirWriteFileOp.resolvePathForStartFile(FSDirWriteFileOp.java:323)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInt(FSNamesystem.java:2374)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFile(FSNamesystem.java:2318)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.create(NameNodeRpcServer.java:771)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.create(ClientNamenodeProtocolServerSideTranslatorPB.java:451)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:523)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:991)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:869)
	at org.apache.hadoop.ipc.Server$RpcCall.run(Server.java:815)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1875)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2675)
```

## 3.6.2 解决: 参考：https://www.ibm.com/docs/en/watson-studio-local/1.2.1?topic=sucdhcwdl-protect-hive-data-access-it-securely-from-data-science-experience-local
```
hdfs dfs -setfacl -R -m user:livy:rwx /griffin/persist;
hdfs dfs -getfacl /griffin/persist;

```

# -------------------------------------------------------------------
## 3.7.1 livy 报错类似如下
```
org.apache.spark.sql.AnalysisException: org.apache.hadoop.hive.ql.metadata.HiveException: Unable to alter table. User livy does not have privileges for ALTERTABLE_ADDCOLS;
	at org.apache.spark.sql.hive.HiveExternalCatalog.withClient(HiveExternalCatalog.scala:108)
	at org.apache.spark.sql.hive.HiveExternalCatalog.alterTableDataSchema(HiveExternalCatalog.scala:687)
	at org.apache.spark.sql.catalyst.catalog.ExternalCatalogWithListener.alterTableDataSchema(ExternalCatalogWithListener.scala:132)
	at org.apache.spark.sql.catalyst.catalog.SessionCatalog.alterTableDataSchema(SessionCatalog.scala:386)
	at org.apache.spark.sql.hive.HiveMetastoreCatalog.updateDataSchema(HiveMetastoreCatalog.scala:264)
	at org.apache.spark.sql.hive.HiveMetastoreCatalog.org$apache$spark$sql$hive$HiveMetastoreCatalog$$inferIfNeeded(HiveMetastoreCatalog.scala:248)
	at org.apache.spark.sql.hive.HiveMetastoreCatalog$$anonfun$4$$anonfun$5.apply(HiveMetastoreCatalog.scala:167)
	at org.apache.spark.sql.hive.HiveMetastoreCatalog$$anonfun$4$$anonfun$5.apply(HiveMetastoreCatalog.scala:156)
```

## 3.7.2 解决: 参考：https://stackoverflow.com/questions/57821080/user-does-not-have-privileges-for-altertable-addcols-while-using-spark-sql-to-re
```
spark.conf.set("spark.sql.hive.caseSensitiveInferenceMode", "INFER_ONLY")

# 即在“spark-conf/spark-defaults.conf 的 Spark 客户端高级配置代码段（安全阀）”中添加
spark.sql.hive.caseSensitiveInferenceMode=INFER_ONLY

# 最后，该参数所有配置类似如下：
spark.master=yarn
spark.yarn.jars=hdfs://odtscluster/user/spark/jars/*.jar
spark.yarn.dist.files=hdfs://odtscluster/user/spark/spark_conf/hive-site.xml
spark.sql.broadcastTimeout=500
spark.yarn.maxAppAttempts=3
spark.yarn.am.attemptFailuresValidityInterval=1h
spark.sql.hive.caseSensitiveInferenceMode=INFER_ONLY
```

### livy 全部配置OK后，最后运行任务的 web 日志类似如下：
```
      "data.time.zone" : "",
      "config" : {
        "database" : "bigdata",
        "table.name" : "griffin_test_pt_tgt",
        "where" : "dt=20210608"
      }
    },
    "baseline" : false
  } ],
  "evaluate.rule" : {
    "id" : 2,
    "rules" : [ {
      "id" : 3,
      "rule" : "source.id=target.id",
      "dsl.type" : "griffin-dsl",
      "dq.type" : "ACCURACY",
      "out.dataframe.name" : "accuracy"
    } ]
  },
  "measure.type" : "griffin"
}
21/06/09 11:15:34 INFO conf.HiveConf: Found configuration file file:/run/cloudera-scm-agent/process/6499-livy-LIVY_REST_SERVER/hive-conf/hive-site.xml
21/06/09 11:15:34 INFO spark.SparkContext: Running Spark version 2.4.0-cdh6.3.2
21/06/09 11:15:34 INFO logging.DriverLogger: Added a local log appender at: /tmp/spark-0cc0ca0c-8311-4ca3-bf14-a5322fb3aaa5/__driver_logs__/driver.log
21/06/09 11:15:34 INFO spark.SparkContext: Submitted application: griffin_test_src_tgt_id_check_job
21/06/09 11:15:34 INFO spark.SecurityManager: Changing view acls to: livy
21/06/09 11:15:34 INFO spark.SecurityManager: Changing modify acls to: livy
21/06/09 11:15:34 INFO spark.SecurityManager: Changing view acls groups to: 
21/06/09 11:15:34 INFO spark.SecurityManager: Changing modify acls groups to: 
21/06/09 11:15:34 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(livy); groups with view permissions: Set(); users  with modify permissions: Set(livy); groups with modify permissions: Set()
21/06/09 11:15:34 INFO util.Utils: Successfully started service 'sparkDriver' on port 33148.
21/06/09 11:15:34 INFO spark.SparkEnv: Registering MapOutputTracker
21/06/09 11:15:34 INFO spark.SparkEnv: Registering BlockManagerMaster
21/06/09 11:15:34 INFO storage.BlockManagerMasterEndpoint: Using org.apache.spark.storage.DefaultTopologyMapper for getting topology information
21/06/09 11:15:34 INFO storage.BlockManagerMasterEndpoint: BlockManagerMasterEndpoint up
21/06/09 11:15:34 INFO storage.DiskBlockManager: Created local directory at /tmp/blockmgr-baad2513-86e5-44d4-89d4-6b7b6e74d728
21/06/09 11:15:34 INFO memory.MemoryStore: MemoryStore started with capacity 366.3 MB
21/06/09 11:15:34 INFO spark.SparkEnv: Registering OutputCommitCoordinator
21/06/09 11:15:34 INFO util.log: Logging initialized @2654ms
21/06/09 11:15:34 INFO server.Server: jetty-9.3.z-SNAPSHOT, build timestamp: 2018-09-05T05:11:46+08:00, git hash: 3ce520221d0240229c862b122d2b06c12a625732
21/06/09 11:15:34 INFO server.Server: Started @2717ms
21/06/09 11:15:34 INFO server.AbstractConnector: Started ServerConnector@214beff9{HTTP/1.1,[http/1.1]}{0.0.0.0:4040}
21/06/09 11:15:34 INFO util.Utils: Successfully started service 'SparkUI' on port 4040.
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@429aeac1{/jobs,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@5ccb85d6{/jobs/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@d88f893{/jobs/job,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@48eaf42f{/jobs/job/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@488f3dd1{/stages,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2091833{/stages/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@7bc58891{/stages/stage,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@1f43cab7{/stages/stage/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@481558ce{/stages/pool,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2668c286{/stages/pool/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@7f353a0f{/storage,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@7ea2412c{/storage/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@1c93b51e{/storage/rdd,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@7c0f28f8{/storage/rdd/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@2abc8034{/environment,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@467b0f6e{/environment/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@474179fa{/executors,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@420cd102{/executors/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@c7f4457{/executors/threadDump,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@12e12ac9{/executors/threadDump/json,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@5e230fc6{/static,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@32f5ecc4{/,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@19bedeb8{/api,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@115ca7de{/jobs/job/kill,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@29fe4840{/stages/stage/kill,null,AVAILABLE,@Spark}
21/06/09 11:15:34 INFO ui.SparkUI: Bound SparkUI to 0.0.0.0, and started at http://onedts-dev-cdh-master-v01.smartparkos.com:4040
21/06/09 11:15:34 INFO spark.SparkContext: Added JAR hdfs:///griffin/jars/griffin-measure.jar at hdfs:///griffin/jars/griffin-measure.jar with timestamp 1623208534954
21/06/09 11:15:34 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:35 INFO security.YARNHadoopDelegationTokenManager: Attempting to load user's ticket cache.
21/06/09 11:15:35 INFO security.HadoopFSDelegationTokenProvider: getting token for: DFS[DFSClient[clientName=DFSClient_NONMAPREDUCE_1152113719_1, ugi=livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM (auth:KERBEROS)]] with renewer yarn/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
21/06/09 11:15:35 INFO hdfs.DFSClient: Created token for livy: HDFS_DELEGATION_TOKEN owner=livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM, renewer=yarn, realUser=, issueDate=1623208535082, maxDate=1623813335082, sequenceNumber=9559, masterKeyId=259 on ha-hdfs:odtscluster
21/06/09 11:15:35 INFO security.HadoopFSDelegationTokenProvider: getting token for: DFS[DFSClient[clientName=DFSClient_NONMAPREDUCE_1152113719_1, ugi=livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM (auth:KERBEROS)]] with renewer livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM
21/06/09 11:15:35 INFO hdfs.DFSClient: Created token for livy: HDFS_DELEGATION_TOKEN owner=livy/onedts-dev-cdh-master-v01.smartparkos.com@SMARTPARKOS.COM, renewer=livy, realUser=, issueDate=1623208535088, maxDate=1623813335088, sequenceNumber=9560, masterKeyId=259 on ha-hdfs:odtscluster
21/06/09 11:15:35 INFO security.HadoopFSDelegationTokenProvider: Renewal interval is 86400030 for token HDFS_DELEGATION_TOKEN
21/06/09 11:15:35 INFO Configuration.deprecation: yarn.resourcemanager.zk-address is deprecated. Instead, use hadoop.zk.address
21/06/09 11:15:35 INFO Configuration.deprecation: yarn.resourcemanager.system-metrics-publisher.enabled is deprecated. Instead, use yarn.system-metrics-publisher.enabled
21/06/09 11:15:35 INFO deploy.SparkHadoopUtil: Updating delegation tokens for current user.
21/06/09 11:15:35 INFO util.Utils: Using initial executors = 2, max of spark.dynamicAllocation.initialExecutors, spark.dynamicAllocation.minExecutors and spark.executor.instances
21/06/09 11:15:35 INFO yarn.Client: Requesting a new application from cluster with 3 NodeManagers
21/06/09 11:15:35 INFO conf.Configuration: resource-types.xml not found
21/06/09 11:15:35 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
21/06/09 11:15:35 INFO yarn.Client: Verifying our application has not requested more than the maximum memory capability of the cluster (8192 MB per container)
21/06/09 11:15:35 INFO yarn.Client: Will allocate AM container, with 896 MB memory including 384 MB overhead
21/06/09 11:15:35 INFO yarn.Client: Setting up container launch context for our AM
21/06/09 11:15:35 INFO yarn.Client: Setting up the launch environment for our AM container
21/06/09 11:15:35 INFO yarn.Client: Preparing resources for our AM container
21/06/09 11:15:35 INFO yarn.Client: Source and destination file systems are the same. Not copying hdfs:/user/spark/spark_conf/hive-site.xml
21/06/09 11:15:35 INFO yarn.Client: Uploading resource file:/tmp/spark-0cc0ca0c-8311-4ca3-bf14-a5322fb3aaa5/__spark_conf__5900348278189846336.zip -> hdfs://odtscluster/user/livy/.sparkStaging/application_1623204499124_0023/__spark_conf__.zip
21/06/09 11:15:35 INFO spark.SecurityManager: Changing view acls to: livy
21/06/09 11:15:35 INFO spark.SecurityManager: Changing modify acls to: livy
21/06/09 11:15:35 INFO spark.SecurityManager: Changing view acls groups to: 
21/06/09 11:15:35 INFO spark.SecurityManager: Changing modify acls groups to: 
21/06/09 11:15:35 INFO spark.SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(livy); groups with view permissions: Set(); users  with modify permissions: Set(livy); groups with modify permissions: Set()
21/06/09 11:15:35 INFO yarn.Client: Submitting application application_1623204499124_0023 to ResourceManager
21/06/09 11:15:35 INFO impl.YarnClientImpl: Submitted application application_1623204499124_0023
21/06/09 11:15:35 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:36 INFO yarn.Client: Application report for application_1623204499124_0023 (state: ACCEPTED)
21/06/09 11:15:36 INFO yarn.Client: 
	 client token: Token { kind: YARN_CLIENT_TOKEN, service:  }
	 diagnostics: AM container is launched, waiting for AM container to Register with RM
	 ApplicationMaster host: N/A
	 ApplicationMaster RPC port: -1
	 queue: root.users.livy
	 start time: 1623208535585
	 final status: UNDEFINED
	 tracking URL: http://onedts-dev-cdh-master-v01.smartparkos.com:8088/proxy/application_1623204499124_0023/
	 user: livy
21/06/09 11:15:36 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:37 INFO yarn.Client: Application report for application_1623204499124_0023 (state: ACCEPTED)
21/06/09 11:15:37 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:38 INFO cluster.YarnClientSchedulerBackend: Add WebUI Filter. org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter, Map(PROXY_HOSTS -> onedts-dev-cdh-master-v01.smartparkos.com,onedts-dev-cdh-master-v02.smartparkos.com, PROXY_URI_BASES -> http://onedts-dev-cdh-master-v01.smartparkos.com:8088/proxy/application_1623204499124_0023,http://onedts-dev-cdh-master-v02.smartparkos.com:8088/proxy/application_1623204499124_0023, RM_HA_URLS -> onedts-dev-cdh-master-v01.smartparkos.com:8088,onedts-dev-cdh-master-v02.smartparkos.com:8088), /proxy/application_1623204499124_0023
21/06/09 11:15:38 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /jobs, /jobs/json, /jobs/job, /jobs/job/json, /stages, /stages/json, /stages/stage, /stages/stage/json, /stages/pool, /stages/pool/json, /storage, /storage/json, /storage/rdd, /storage/rdd/json, /environment, /environment/json, /executors, /executors/json, /executors/threadDump, /executors/threadDump/json, /static, /, /api, /jobs/job/kill, /stages/stage/kill.
21/06/09 11:15:38 INFO yarn.Client: Application report for application_1623204499124_0023 (state: RUNNING)
21/06/09 11:15:38 INFO yarn.Client: 
	 client token: Token { kind: YARN_CLIENT_TOKEN, service:  }
	 diagnostics: N/A
	 ApplicationMaster host: 192.168.100.106
	 ApplicationMaster RPC port: -1
	 queue: root.users.livy
	 start time: 1623208535585
	 final status: UNDEFINED
	 tracking URL: http://onedts-dev-cdh-master-v01.smartparkos.com:8088/proxy/application_1623204499124_0023/
	 user: livy
21/06/09 11:15:38 INFO cluster.YarnClientSchedulerBackend: Application application_1623204499124_0023 has started running.
21/06/09 11:15:38 INFO cluster.SchedulerExtensionServices: Starting Yarn extension services with app application_1623204499124_0023 and attemptId None
21/06/09 11:15:38 INFO util.Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 45941.
21/06/09 11:15:38 INFO netty.NettyBlockTransferService: Server created on onedts-dev-cdh-master-v01.smartparkos.com:45941
21/06/09 11:15:38 INFO storage.BlockManager: Using org.apache.spark.storage.RandomBlockReplicationPolicy for block replication policy
21/06/09 11:15:38 INFO storage.BlockManagerMaster: Registering BlockManager BlockManagerId(driver, onedts-dev-cdh-master-v01.smartparkos.com, 45941, None)
21/06/09 11:15:38 INFO storage.BlockManagerMasterEndpoint: Registering block manager onedts-dev-cdh-master-v01.smartparkos.com:45941 with 366.3 MB RAM, BlockManagerId(driver, onedts-dev-cdh-master-v01.smartparkos.com, 45941, None)
21/06/09 11:15:38 INFO storage.BlockManagerMaster: Registered BlockManager BlockManagerId(driver, onedts-dev-cdh-master-v01.smartparkos.com, 45941, None)
21/06/09 11:15:38 INFO storage.BlockManager: external shuffle service port = 7337
21/06/09 11:15:38 INFO storage.BlockManager: Initialized BlockManager: BlockManagerId(driver, onedts-dev-cdh-master-v01.smartparkos.com, 45941, None)
21/06/09 11:15:38 INFO ui.JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /metrics/json.
21/06/09 11:15:38 INFO handler.ContextHandler: Started o.s.j.s.ServletContextHandler@5e193ef5{/metrics/json,null,AVAILABLE,@Spark}
21/06/09 11:15:38 INFO scheduler.EventLoggingListener: Logging events to hdfs://odtscluster/user/spark/applicationHistory/application_1623204499124_0023
21/06/09 11:15:38 INFO util.Utils: Using initial executors = 2, max of spark.dynamicAllocation.initialExecutors, spark.dynamicAllocation.minExecutors and spark.executor.instances
21/06/09 11:15:38 WARN cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: Attempted to request executors before the AM has registered!
21/06/09 11:15:38 WARN lineage.LineageWriter: Lineage directory /var/log/spark/lineage doesn't exist or is not writable. Lineage for this application will be disabled.
21/06/09 11:15:38 INFO util.Utils: Extension com.cloudera.spark.lineage.NavigatorAppListener not being initialized.
21/06/09 11:15:38 INFO logging.DriverLogger$DfsAsyncWriter: Started driver log file sync to: /user/spark/driverLogs/application_1623204499124_0023_driver.log
21/06/09 11:15:38 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:39 INFO cluster.YarnSchedulerBackend$YarnSchedulerEndpoint: ApplicationMaster registered as NettyRpcEndpointRef(spark-client://YarnAM)
21/06/09 11:15:39 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:40 INFO yarn.SparkRackResolver: Got an error when resolving hostNames. Falling back to /default-rack for all
21/06/09 11:15:41 INFO cluster.YarnSchedulerBackend$YarnDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (192.168.100.106:51064) with ID 2
21/06/09 11:15:41 INFO spark.ExecutorAllocationManager: New executor 2 has registered (new total is 1)
21/06/09 11:15:41 INFO cluster.YarnSchedulerBackend$YarnDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (192.168.100.106:51070) with ID 1
21/06/09 11:15:41 INFO spark.ExecutorAllocationManager: New executor 1 has registered (new total is 2)
21/06/09 11:15:41 INFO storage.BlockManagerMasterEndpoint: Registering block manager onedts-dev-cdh-datanode-v01.smartparkos.com:37723 with 366.3 MB RAM, BlockManagerId(2, onedts-dev-cdh-datanode-v01.smartparkos.com, 37723, None)
21/06/09 11:15:41 INFO cluster.YarnClientSchedulerBackend: SchedulerBackend is ready for scheduling beginning after reached minRegisteredResourcesRatio: 0.8
21/06/09 11:15:41 INFO storage.BlockManagerMasterEndpoint: Registering block manager onedts-dev-cdh-datanode-v01.smartparkos.com:37424 with 366.3 MB RAM, BlockManagerId(1, onedts-dev-cdh-datanode-v01.smartparkos.com, 37424, None)
21/06/09 11:15:42 WARN lineage.LineageWriter: Lineage directory /var/log/spark/lineage doesn't exist or is not writable. Lineage for this application will be disabled.
21/06/09 11:15:43 INFO measure.Application$: process init success
21/06/09 11:15:47 INFO datasource.DataSource: load data [source]
21/06/09 11:15:47 INFO batch.HiveBatchDataConnector: SELECT * FROM bigdata.griffin_test_pt_src WHERE dt=20210608
21/06/09 11:15:48 INFO datasource.DataSource: load data [target]
21/06/09 11:15:48 INFO batch.HiveBatchDataConnector: SELECT * FROM bigdata.griffin_test_pt_tgt WHERE dt=20210608
data source timeRanges: source -> (1623122131657, 1623122131657], target -> (1623122131657, 1623122131657]
21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: main begin transform step : 
accuracy
|   |---__totalCount
|   |---__missCount
|   |   |---__missRecords

21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: transform-step-0 begin transform step : 
__totalCount

21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: transform-step-1 begin transform step : 
__missCount
|   |---__missRecords

21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: transform-step-3 begin transform step : 
__missRecords

21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: transform-step-0 end transform step : 
__totalCount

21/06/09 11:15:48 INFO transform.SparkSqlTransformStep: transform-step-3 end transform step : 
__missRecords

21/06/09 11:15:48 INFO context.DataFrameCache: try to cache data frame __missRecords
21/06/09 11:15:48 INFO context.DataFrameCache: cache after replace no old df
21/06/09 11:15:51 INFO transform.SparkSqlTransformStep: transform-step-1 end transform step : 
__missCount
|   |---__missRecords

21/06/09 11:15:51 INFO transform.SparkSqlTransformStep: main end transform step : 
accuracy
|   |---__totalCount
|   |---__missCount
|   |   |---__missRecords

21/06/09 11:15:52 INFO sink.SinkTaskRunner$: task 1623122131657 success with (true) [ using time 211 ms ]
21/06/09 11:15:52 INFO apache.griffin: Time taken: 9121 milliseconds
21/06/09 11:15:52 INFO measure.Application$: process run result: success
21/06/09 11:15:52 INFO measure.Application$: process end success
```
