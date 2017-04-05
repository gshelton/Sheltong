## Configure SSL between Ranger and Rager HDFS plugin with CA Signed certs

# Short Description:

Configure SSL between Ranger and Rager HDFS plugin with CA Signed certs


Ranger communicates with Plug-ins only with 2 WAY SSL (1 way SSL in not allowed).

[Updated] Appears like one way SSL is possible with latest patch - https://issues.apache.org/jira/browse/RANGER-1094

First get server keystore as skeystore.jks and truststore strustore.jks and client keystore as ckeystore.jks and ctruststore.jks (you can create these keystore/truststores once you get the signed certs from CA Signing.

Here is the steps:

1. Login to Ambari
Go to Ranger > Configs > Ranger Settings > External URL points to a URL that uses SSL: https://<hostname of Ranger>:<https port, default is 6182> and ranger.service.https.attrib.ssl.enabled to false

2. Go to HDFS > Configs > Advanced > ranger-hdfs-policymgr-ssl and set the following properties:
    xasecure.policymgr.clientssl.keystore = /etc/hadoop/conf/ckeystore.jks
    xasecure.policymgr.clientssl.keystore.password = bigdata 
    xsecure.policymgr.clientssl.truststore = strustore.jks
    xasecure.policymgr.clientssl.truststore.password = bigdata
3. Go to HDFS > Configs > Advanced > Advanced ranger-hdfs-plugin-properties
     common.name.for.certificate = specify the common name (or alias) that is specified in ckeystore.jks 
4.HDFS > Configs > Advanced > Advanced ranger-hdfs-plugin-properties then select the Enable Ranger for HDFS check box. 

5.Go to  Ranger > Configs > Ranger Settings > Advanced ranger-admin-site
    ranger.https.attrib.keystore.file=skeystore.jks
    ranger.service.https.attrib.keystore.pass=bigdata
    ranger.service.https.attrib.keystore.keyalias=specify alias name that is specified in skeystore.jks file 
    ranger.service.https.attrib.clientAuth=want
 
Add below under custom Ranger-admin-site
    ranger.service.https.attrib.client.auth=want
    ranger.service.https.attrib.keystore.file=skeystore.jks

6.Log into the Ranger Policy Manager UI as the admin user. Click the Edit button of your repository (in this case, hadoopdev) and provide the CN name of the keystore as the value for Common Name For Certificate, then save your changes.

7. This is applicable only for HDP2.5 (this is a bug 2.5 hence modifying the sh script)

Go to /usr/hdp/current/ranger-admin/ews/ranger-admin-services.sh
Edit the JAVA_OPTS to add trustore and truststorepassword
 
    JAVA_OPTS=" ${JAVA_OPTS} -XX:MaxPermSize=256m -Xmx1024m -Xms1024m -Djavax.net.ssl.trustStore=/tmp/rangercerts/ctruststore.jks -Djavax.net.ssl.trustStorePassword=bigdata"

8. Restart all the service and you HDFS plug-in should be able to communicate with Ranger service.

Note:

while creating the client certs, make sure you provide extension as "usr_cert" and server cert as "server_cert" , other wise 2 WAY SSL communication would fail.