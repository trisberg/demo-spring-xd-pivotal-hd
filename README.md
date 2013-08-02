Demo using Spring XD with Pivotal HD
====================================

Step 1: Download, install and start Pivotal HD VM
-------------------------------------------------

Instructions for downloading and starting the VM: 
http://pivotalhd.cfapps.io/getting-started/pivotalhd-vm.html

Step 2: Download Spring XD 1.0 M2 release
-----------------------------------------

From the Pivotal HD VM download Spring XD 1.0 M2 release using this link: 
http://www.springsource.org/download/community?project=Spring%20XD

Step 3: Unzip and start Spring XD 1.0 M2 release
------------------------------------------------

Unzip the `spring-xd-1.0.0.M2.zip` file into a `/home/gpadmin/spring-xd-1.0.0.M2` directory.

Open a command prompt and enter the following commands:

   export XD_HOME=/home/gpadmin/spring-xd-1.0.0.M2/xd
   $XD_HOME/bin/xd-singlenode --hadoopDistro phd1 --httpPort8090

We need to specify the hadoop distro as phd1 since we are running against Pivotal HD and we also 
need to specify the HTTP port since port 8080 is already in use on this VM.



