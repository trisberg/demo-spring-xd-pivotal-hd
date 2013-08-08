# Demo using Spring XD with Pivotal HD

## Preparing the VM

### Step 1: Download, install and start Pivotal HD VM

Instructions for downloading and starting the VM: 
http://pivotalhd.cfapps.io/getting-started/pivotalhd-vm.html

### Step 2: Download Spring XD 1.0 M2 release

From the Pivotal HD VM download Spring XD 1.0 M2 release using this link: 
http://www.springsource.org/download/community?project=Spring%20XD

### Step 3: Unzip and start Spring XD 1.0 M2 release

Unzip the `spring-xd-1.0.0.M2.zip` file into the home directory of the gpadmin user. This should create 
a `/home/gpadmin/spring-xd-1.0.0.M2` directory.

Open a command prompt and enter the following commands:

    export XD_HOME=/home/gpadmin/spring-xd-1.0.0.M2/xd
    cd $XD_HOME
    ./bin/xd-singlenode --hadoopDistro phd1 --httpPort 8090

We need to specify the hadoop distro as phd1 since we are running against Pivotal HD and we also 
need to specify the HTTP port since port 8080 is already in use on this VM.

### Step 4: Start the Spring XD shell

Open another command prompt and enter the following command:

    /home/gpadmin/spring-xd-1.0.0.M2/shell/bin/spring-xd-shell --hadoopDistro phd1
    
Once the shell starts up we need to connect to the Spring XD admin server:

    server-unknown:> admin config server --uri http://localhost:8090

To be able to run Hadoop fs commands from the shell, we also need to set the hdfs configuration:

    xd:> hadoop config fs --namenode hdfs://pivhdsne:8020
    

## The Demos

### Demo1 - Twitter search using HAWQ external table in hdfs

*This is a Spring XD "twittersearch | transform | hdfs" stream.*

In order to query the Twitter data with an HAWQ external table we need to provide the Twitter data in a 
tab-delimited format with one line per row. For this demo we will be querying the hash tags so we will create one 
line per hash tag, so data will be de-normalized since tweets with multiple hash tags will have additional rows. 
The easiest way to do this reformatting is is to provide a transformer script.We have written one in Groovy that can 
be viewed or downloaded here: 
[tweets-delim.groovy](https://raw.github.com/trisberg/demo-spring-xd-pivotal-hd/master/modules/processor/scripts/tweets-delim.groovy)

To use this script in our XD stream we need to copy it to the Spring XD `modules/processor/scripts` directory. We can do that 
by opening another command prompt and entering the following commwnd:

    wget -O /home/gpadmin/spring-xd-1.0.0.M2/xd/modules/processor/scripts/tweets-delim.groovy https://raw.github.com/trisberg/demo-spring-xd-pivotal-hd/master/modules/processor/scripts/tweets-delim.groovy 

We also need to modify the connection properties for HDFS which are specified in `config/hadoop.properties`. We can edit that file using this command:

    gedit /home/gpadmin/spring-xd-1.0.0.M2/xd/config/hadoop.properties

Then change the content of the file to the following:

```
fs.defaultFS=hdfs://pivhdsne:8020
```

Last config task is to add your Twitter consumerKey and consumerSecret to `config/twitter.properties`. We can edit that file using this command:

    gedit /home/gpadmin/spring-xd-1.0.0.M2/xd/config/twitter.properties
    
See the [Spring XD docs](https://github.com/SpringSource/spring-xd/wiki/Sources#wiki-twittersearch) for more details.

We are now ready to create the stream, so we switch back to the Spring XD shell:

    xd:> stream create --name tweets --definition "twittersearch --query='hadoop' | transform --script=tweets-delim.groovy | hdfs --rollover=10000"

We should see the stream get created in the Spring XD admin window. From the shell we can list the streams using:

    xd:> stream list
    
We let this stream run for a little while to collect some tweets. We can check that we actually have a data file created
in hdfs by entering the following command in the Spring XDshell:

    xd:> hadoop fs ls /xd/tweets

if the `tweets-0.log` file has 0 size it just means that our rollover limit has not been reached yet.

We can stop the stream to flush all the data using:

    xd:> stream undeploy --name tweets
    
Now we can look at the content in the file in hdfs using:

    xd:> hadoop fs cat /xd/tweets/tweets-0.log
    
So far so good. Next step is to import this data into HAWQ using the PXF External Tables feature. We open up a new command window
and enter `psql` to start a PostgreSQL client shell. We are automatically logged in as gpadmin. Create the external table using the 
following command:

     CREATE EXTERNAL TABLE tweets(
       id BIGINT, from_user VARCHAR(255), created_at TIMESTAMPTZ, hash_tag VARCHAR(255), 
       followers INTEGER, language_code VARCHAR(10), retweet_count INTEGER, retweet BOOLEAN) 
     LOCATION ('pxf://pivhdsne:50070/xd/tweets/*.log?Fragmenter=HdfsDataFragmenter&Accessor=TextFileAccessor&Resolver=TextResolver') 
     FORMAT 'TEXT' (DELIMITER = E'\t');

Once the table is created we can query it:

    select count(*) from tweets;
     count
    -------
        85
    (1 row)
     
We can also run the following query to get the hash tags that were used most often:

    select lower(hash_tag) as hash_tag, count(*) 
    from tweets where hash_tag != '-' 
    group by lower(hash_tag) 
    order by count(*) desc limit 10;


### Demo2 - Twitter search storing data directly into HAWQ using JDBC

*This is a Spring XD "twittersearch | jdbc" stream.*

For this demo we will be querying Twitter for tweets matching our query and then storing some of the fields in a table in
HAWQ. We will insrt one row per tweet using the Spring XD JDBC sink.

We need to modify the connection properties for JDBC which are specified in `config/jdbc.properties`. We can edit that file using this command:

    gedit /home/gpadmin/spring-xd-1.0.0.M2/xd/config/jdbc.properties

Then change the content of the file to the following:

```
driverClass=org.postgresql.Driver
url=jdbc:postgresql:gpadmin
username=gpadmin
password=
initializeDatabase=false
```

Next step is to create the table in HAWQ. We open up a new command window and enter `psql` to start a PostgreSQL client 
shell. We are automatically logged in as gpadmin. Create the external table using the following command:

     CREATE TABLE jdbc_tweets(
       id BIGINT, from_user VARCHAR(255), created_at BIGINT, text VARCHAR(255), 
       followers INTEGER, language_code VARCHAR(10), retweet_count INTEGER, retweet CHAR(5)); 

We don't do any conversion of the incoming data so we need to limit the datatypes used to ones that don't require an explicit cast based on the data 
available in the JSON document we get back from the Twitter search.

Last config task is to add our Twitter consumerKey and consumerSecret to `config/twitter.properties`. We can edit that file using this command:

    gedit /home/gpadmin/spring-xd-1.0.0.M2/xd/config/twitter.properties
    
See the [Spring XD docs](https://github.com/SpringSource/spring-xd/wiki/Sources#wiki-twittersearch) for more details.

We are now ready to create the stream, so we switch back to the Spring XD shell:

    xd:> stream create --name jdbc_tweets --definition "twittersearch --query='hadoop' | jdbc --columns=' id, fromUser, createdAt, text, followers, languageCode, retweetCount, retweet'"

We should see the stream get created in the Spring XD admin window. From the shell we can list the streams using:

    xd:> stream list
    
We let this stream run for a little while to collect some tweets. We can now check that we actually have a some data 
in the table by opening up a new command window and entering `psql` to start a PostgreSQL client shell.
Run the following command to see the data in the table:

    select * from jdbc_tweets;
     
We can stop the stream from the Spring XD Shell using:

    xd:> stream undeploy --name jdbc_tweets

