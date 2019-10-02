# Cheet-Sheet-of-Hadoop-Integration-With-Cloud-Pak-for-Data

## Reference link:

   https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/install/hdp.html#hdp
   https://www.ibm.com/support/knowledgecenter/en/SSAS34_1.2.0/local/hdp.html#create-an-edge-node

## Install Edge node

 1 Edge node hardware requirements
```
   * 8 GB memory
   * 2 CPU cores
   * 100 GB disk, mounted and available on /var in the local Linux file system
   * 10 GB network interface card recommended for multi-tenant environment (1 GB network interface card if WEBHDFS will not be heavily utilized)
```
 2 Create an edge node

The DSXHI service can be installed on a shared edge node if the resources listed above are exclusively available for DSXHI. To create a new edge node, follow the steps in the HDP documentation. When the edge node is successfully created, it should have the following components:
```
   * The Hadoop client installed.
   * The Spark client installed if the HDP cluster has a Spark service.
   * The Spark2 client installed if the HDP cluster has a Spark2 service.
   * For a kerberized cluster, have the spnego keytab copied to /etc/security/keytabs/spnego.service.keytab.
```
  3 Additional prerequisites

   In addition, the following requirements should be met on the edge node:
```
   * Have Python 2.7 installed.
   * Have curl 7.19.7-53 or later to allow secure communication between DSXHI and DSX Local.
   * Have a service user that can run the DSXHI service. This user should be a valid Linux user with a home directory created in HDFS.
   * The service user should have the necessary Hadoop Proxyuser privileges in HDFS, WebHCAT and Livy services to access data and submit asynchronous jobs as DSX Local users.
   * For a kerberized cluster: Have the keytab for the service user. This eliminates the need for every DSX Local user to have a valid keytab.
   * Have an available port for the DSXHI service. The port for DSXHI service should be exposed for access from the DSX Local clusters that need to connect to the HDP cluster.
   * Have an available port for the DSXHI Rest service. This port need not be exposed for external access.
   * Depending on the service that needs to be exposed by DSXHI, have an available port for Livy for Spark and Livy for Spark 2. These ports do not need to be exposed for external access.
   ```

## Install DSXHI in the edge node

1 Get the tar file, untar and install the rpm file (you can run following commands by root):

    wget --no-check-certificate --no-cache -N http://ibm-open-platform.ibm.com/repos/ICP4D/v1.2.1.0/GA/hadoop_integration_software.tar
    tar xvf hadoop_integration_software.tar
    yum install dsxhi-icp4data-dsp-1.1.1.0-64.noarch.rpm
    
2 Create a service account in local linux or in kerberos server and then grant folder permission of /opt/ibm/dsxhi.

    useradd -u 1015 dsxhi
    chown -R dsxhi:dsxhi dsxhi
    
 3 Add service account to sudo permission.
 
   ADD following line to /etc/sudoers in the edge node:
   
    dsxhi ALL=(ALL)       NOPASSWD: ALL
    
 4 Create a /user/<service account> folder in HDFS and grant permission to the service account:
   
   ```
   [hdfs@waspish3 root]$ hadoop fs -ls /user
    Found 9 items
    drwxrwx---   - ambari-qa hdfs           0 2019-09-20 00:00 /user/ambari-qa
    drwxr-xr-x   - dsxhi     dsxhi          0 2019-09-25 13:25 /user/dsxhi
    drwxr-xr-x   - hbase     hdfs           0 2019-09-19 23:51 /user/hbase
    drwxr-xr-x   - hcat      hdfs           0 2019-09-19 23:55 /user/hcat
    drwxr-xr-x   - hive      hdfs           0 2019-09-19 23:55 /user/hive
    drwxrwxr-x   - livy      hdfs           0 2019-09-24 20:44 /user/livy
    drwxrwxr-x   - oozie     hdfs           0 2019-09-19 23:57 /user/oozie
    drwxr-xr-x   - root      root           0 2019-09-25 13:16 /user/root
    drwxrwxr-x   - spark     hdfs           0 2019-09-19 23:52 /user/spark
   ```
  
  5 Add the service account to Hadoop Proxyuser privileges.
  
    Go to HDP ambari web console, then choose service->hive->custom webhcat-site:
   
    Add webhcat.proxyuser.dsxhi.groups and webhcat.proxyuser.dsxhi.hosts in the setting:
    
   ![Hive configuration](hive-webcat-dsxhi.jpg?raw=true)
  
  6 For a kerberized cluster: Have the keytab for the service user. 
  
   After creating the service user in kerveros server, using similar commands as below to generate the keytab file:
  ``` 
   # ktutil:
   # Run the below commands at ktutil prompt 
   addent -password -p mongodb/mdb01.mdbkrb5.net -k 2 -e aes256-cts
   # Password for mongodb/mdb01.mdbkrb5.net@MDBKRB5.NET: 
   write_kt /var/lib/mongo/private/mon01.keytab
   ```
 ## Switch to service user and Config /opt/ibm/dsxhi/conf/dsxhi_install.conf
 
  Here is a example configuration file:
  
 ``` 
    [root@waspish3 ~]# cat /opt/ibm/dsxhi/conf/dsxhi_install.conf
# Optional - Specify 'A' or 'a' for accepting the license.
# Specify 'R' or 'r' for rejecting the license. If the property is
# empty, user will be prompted during installation.
dsxhi_license_acceptance=A

# Mandatory - Specify the username and group of the user (dsxhi service user)
# running the dsxhi service.
dsxhi_serviceuser=dsxhi
dsxhi_serviceuser_group=dsxhi

# If the HDP cluster is kerberized, it is mandatory to specify the complete
# path to the keytab for the dsxhi service user and the spnego keytab.
# If the HDP cluster is not kerberized, this field should be left blank.
dsxhi_serviceuser_keytab=
dsxhi_spnego_keytab=

# Mandatory - Specify the port number for the dsxhi gateway service. This port
# should be accessible externally.
dsxhi_gateway_port=8445

# Mandatory - Specify the port number for the dsxhi rest service.
dsxhi_rest_port=8082

# Optional - Specify the Ambari URL and admin username for the HDP cluster.
# User will be prompted for password during installation. If the url is not
# specified, some pre-checks for the install will not be performed.
cluster_manager_url=http://abroad1.fyre.ibm.com:8081
cluster_admin=admin

# Optional - Specify the hadoop services that dsxhi service should expose.
exposed_hadoop_services=webhdfs,webhcat,livyspark,livyspark2

# Optional - Specify the URL for the webhcat service.
existing_webhcat_url=

# Optional - If the HDP cluster has Livy for SPARK and/or SPARK2 configured,
# then specify the URL.
existing_livyspark_url=http://abroad1.fyre.ibm.com:8998
existing_livyspark2_url=http://abroad1.fyre.ibm.com:8999

# Mandatory - Specify the port number that dsxhi service should use when
# installing and configuring Livy for SPARK and/or SPARK2.
dsxhi_livyspark_port=8998
dsxhi_livyspark2_port=8999

# Optional - Specify the list of URL for the dsx local clusters that will
# register this dsxhi service. The URL should include the port number if
# necessary.
known_dsx_list=https://zen-v2102a-master-1.fyre.ibm.com:31843

# Optional - Provide an installer tool to install packages
# supported tools are yum, rpm, dnf. This option should be set to use the

 ``` 
## Add certificates for all the nodes in Hadoop cluster.

Example command:

    ./add_cert.sh https://abroad1.fyre.ibm.com:50470
    ./add_cert.sh https://abroad2.fyre.ibm.com:50475
    ./add_cert.sh https://abroad3.fyre.ibm.com:50475
    ./add_cert.sh https://abroad4.fyre.ibm.com:50475

## Start installation with service user (root is not accepted) and check the status of dsxhi. 


    /opt/ibm/dsxhi/bin/install.py
    /opt/ibm/dsxhi/bin/status.py
    
   If dsxhi service is not started, using following command to start or stop it:
    
    /opt/ibm/dsxhi/bin/start.py
    /opt/ibm/dsxhi/bin/stop.py
    
    
    
 ## Add a Cloud Pak For Data cluster to the known list 
 
   Example command:
 
    /opt/ibm/dsxhi/bin/manage_known_dsx.py -a "https://zen-v2102a-master-1.fyre.ibm.com:31843"
   
  ## Configure hadoop integration in Cloud Pak For Data   
   
   1 Go to Cloud Pak For Data web console, and choose administrator->configure platform->hadoop integration.
   
   2 Choose "add registration" tab.
   
   3 Input name, URL and service user ID, and you can find the correct service URL from gateway_audit.log:
   
  ```
  
  cat /opt/ibm/dsxhi/gateway/logs/gateway-audit.log | grep gateway
   
   19/09/25 19:11:28 |||audit|172.16.37.100|WEBHDFS|dsxhi|||access|uri|/gateway/zen-v2102a-master-1/webhdfs/data/v1/webhdfs/v1/user/dsxhi/environments/82a58879b4c2158072b3c262577be30a710b33a7c6f79561804f648f66a0e933/dsx-scripted-ml-python36.tar.gz?_=AAAACAAAABAAAACwKaYXJQe3slqLy4BABw-Lnj9V6pW0dvVC1-_p4fnDJ7DnyFKz5FJY-uemEoW_Ey0wV6VzGCDKMmaktG9STEGSWYuyh-l68QlwiKShI18fB5CQRVRNVlQuUxZrczM9tpHShHow2VDPRVWQgA4xnE7enOL-bdVlkJzQwQ7lGrOFmDQKOTTNzXIiHcD4SuDyugH1KFNqX2GRxlDilqZWO3tN55JvEc-9xNbnAXgbR1AwcImnTLkS_cU-0_KsMnv0Vz0nu7MaVKbLn6g|success|Response status: 201

   ```
   
   and here is the example URL:
   
     https://waspish3.fyre.ibm.com:8445/gateway/zen-v2102a-master-1
  
   
## Validate hadoop integration


1 In the hadoop integration page, choose the "details" of the hadoop integration we just added.
2 Make sure you have following endpoints in the details of the integration:

    WebHDFS
    WebHCAT
    Livy for Spark
    Livy for Spark2

3 Create a analytics project and make sure you have data source from hadoop cluster available.

4 Create a notebook in the project and run following command:

   
    import dsx_core_utils
    import pandas as pd
    import numpy as np

    DSXHI_SYSTEMS = dsx_core_utils.get_dsxhi_info(showSummary=False)
    pd.io.json.json_normalize(DSXHI_SYSTEMS, record_path='imageIdNameList', meta=['system']) #[['system','id','name']]
    
 5 Push image to remote hadoop cluster.
 
   
    1) In the details of hadoop integration, go to runtime image and then choose action "push" or "replace image".
   
    2) Monitor the push process by checking the log.
   
    3) Finally you will get the status of the pushing "Push succedded".
    
   ![Push image](hadoop-integration.jpg?raw=true)
   
   
