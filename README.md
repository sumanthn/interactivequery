<h1>Interactive Query With Presto on Parquet Files


----------

Presto(https://prestodb.io/)) is a distributed sql query engine for running interactive queries on large datasets. 
Presto works with many different data sources & supports access to data via SQL. In addition to this, it provides jdbc drivers to allow client to connect.

This repo captures steps to allow for Presto to work with Parquet files
Components & Version
Hadoop - 2.6.2
Hive - 0.14.0
Presto - 0.129
Steps

 - Setup Hadoop & Hive, /usr/local/hadoop & /usr/local/hive
 - Setup Presto 
	 - Within Presto install dir create a file 
   
config.properties

    coordinator=true
    node-scheduler.include-coordinator=true
    http-server.http.port=18080
    query.max-memory=12GB
    query.max-memory-per-node=1GB
    discovery-server.enabled=true
    discovery.uri=http://localhost:28080
 jvm.config

     -server
    -Xmx14G
    -XX:+UseG1GC
    -XX:G1HeapRegionSize=32M
    -XX:+UseGCOverheadLimit
    -XX:+ExplicitGCInvokesConcurrent
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:OnOutOfMemoryError=kill -9 %p

node.properties

    node.environment=production
    node.id=i10
    node.data-dir=/opt/presto/data
Create catalog directory,
create hive.properties

    connector.name=hive-hadoop2
    hive.metastore.uri=thrift://localhost:9083
    hive.config.resources=/usr/local/hadoop/etc/hadoop/conf/core-site.xml,/usr/local/hadoop/etc/hadoop/conf/hdfs-site.xml

Start Hdfs & Hive meta store
Hive meta store 
/usr/local/hive/bin/hive --service metastore
meta store runs on port 9083

Start presto

    /opt/presto/bin/launcher

Parquet Files generator
Use [factdatagenerator](https://github.com/sumanthn/factdatagenerator) to generate Parquet files ,
Example:

     java -Xmx2g -jar target/factdatagenerator.jar Parquet "2015-12-13 00:00:00" "2015-12-14 00:00:00" 1000 /tmp/rawdump/fdata_dec13.parq /usr/local/hadoop/etc/hadoop/conf/

Generates parquet files for 1 day(86.4million records, 2GB)

Create a directory in hdfs

      hdfs df -mkdir  /user/hive/dw/accessdatadump

Copy files to hdfs

    hadoop fs -copyFromLocal /tmp/rawdump/* /user/hive/dw/accessdatadump

Start Hive(required to create table)
Fire create ddl in hive
 

    create table accesslog(
        accessUrl string,
        responseStatusCode int,
        responseTime int,
        accessTimestamp string,
        requestVerb string,
        requestSize int,
        dataExchangeSize int,
        serverIp string,
        clientIp string,
        clientId string,
        sessionId string,
        userAgentDevice string,
        UserAgentType string,
        userAgentFamily string,
        userAgentOSFamily string,
        userAgentVersion string,
        userAgentOSVersion string,
        city string,
        country string,
        region string,
        minOfDay int,
        hourOfDay int,
        dayOfWeek int,
        monthOfYear int
        )stored as parquet LOCATION '/user/hive/dw/accessdatadump';

Shutdown Hive(Presto doesn't use hive for execution, it requires metastore service to infer schema and data location)

<h2>Testing 

Download command line running from presto
Connect to presto 

    ./presto --server localhost:28080 --catalog hive --schema default

Fire queries from command prompt

Connecting via JDBC driver

     Class.forName("com.facebook.presto.jdbc.PrestoDriver");
                //jdbc:presto://localhost:28080/hive/default
                Connection connection = DriverManager.getConnection("jdbc:presto://localhost:28080/hive/default", "test", null);
                Statement statement = connection.createStatement();
                ResultSet rs = statement.executeQuery("select * from accesslog");
                while (rs.next()) {
                    System.out.println(rs.getString(1));
                }
     statement.close();
                connection.close();
            }catch (Exception e){
                e.printStackTrace();
            }

TODO:

 - Work on Partitioned data  
 - Work with in S3 bucket





