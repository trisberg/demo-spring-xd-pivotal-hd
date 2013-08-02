# Demo using Spring XD with Pivotal HD

## Preparing the VM

### Step 1: Download, install and start Pivotal HD VM

Instructions for downloading and starting the VM: 
http://pivotalhd.cfapps.io/getting-started/pivotalhd-vm.html

### Step 2: Download Spring XD 1.0 M2 release

From the Pivotal HD VM download Spring XD 1.0 M2 release using this link: 
http://www.springsource.org/download/community?project=Spring%20XD

### Step 3: Unzip and start Spring XD 1.0 M2 release

Unzip the `spring-xd-1.0.0.M2.zip` file into a `/home/gpadmin/spring-xd-1.0.0.M2` directory.

Open a command prompt and enter the following commands:

    export XD_HOME=/home/gpadmin/spring-xd-1.0.0.M2/xd
    $XD_HOME/bin/xd-singlenode --hadoopDistro phd1 --httpPort 8090

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
tab-delimited format with one line per tweet. The easiest way to do that is to provide a transformer script.
We have written one in Groovy that can be viewed or downloaded here: 
[tweets-delim.groovy](https://raw.github.com/trisberg/demo-spring-xd-pivotal-hd/master/modules/processor/scripts/tweets-delim.groovy)

To use this script in our XD stream we need to copy it to the Spring XD `modules/processor/scripts` directory. We can do that 
by opening another command prompt and entering the following commwnd:

    wget -O /home/gpadmin/spring-xd-1.0.0.M2/xd/modules/processor/scripts/tweets-delim.groovy https://raw.github.com/trisberg/demo-spring-xd-pivotal-hd/master/modules/processor/scripts/tweets-delim.groovy 

We also need to modify the connection properties for HDFS which are specified in `config\hadoop.properties`. We can edit that file using this command:

    gedit /home/gpadmin/pring-xd-1.0.0.M2/xd/config/hadoop.properties

Then change the content of the file to the following:

```
fs.default.name=hdfs://pivhdsne:8020
```

We are now ready to create the stream, so we switch back to the Spring XD shell.
