## Configure two Kerberos KDCs as a Master/Slave


Use a single Kerberos realm for multiple HDP clusters through the user of a Master/Slave configuration
Article

The setup of two separate KDCs in a Master/Slave configuration. 

This setup will allow two clusters to share a single Kerberos realm, which allows the principals to be recognized between clusters. A use case for this configuration is when a Disaster Recovery cluster is used as a warm standby. 

The high level information for the article was found at https://web.mit.edu/kerberos/krb5-1.13/doc/admin/install_kdc.html, while the details were worked out through sweat and tears.

Execute the following command to install the Master and Slave KDC if the KDC is not already installed:

yum install krb5-server 
The following defines the KDC configuration for both clusters. This file, /etc/krb5.conf, must be copied to each node in the cluster.

[libdefaults]
  renew_lifetime = 7d
  forwardable = true
  default_realm = CUSTOMER.HDP
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false
  udp_preference_limit=1
[domain_realm]
  customer.com = CUSTOMER.HDP
  .customer.com = CUSTOMER.HDP
[logging]
  default = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log
  kdc = FILE:/var/log/krb5kdc.log
[realms]
  CUSTOMER.HDP = {
    admin_server = master-kdc.customer.com
    kdc = master-kdc.customer.com
    kdc = slave-kdc.customer.com
  }


Contents of /var/kerberos/krb5kdc/kadm5.acl:

*/admin@CUSTOMER.HDP *
Contents of the /var/kerberos/krb5kdc/kdc.conf:

[kdcdefaults]
 kdc_ports = 88,750
 kdc_tcp_ports = 88,750
[realms]
    CUSTOMER.HDP = {
        kadmind_port = 749
        max_life = 12h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        master_key_type = aes256-cts
       supported_enctypes = aes256-cts:normal aes128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal arcfour-hmac:normal des-cbc-md5:normal
    }
Contents of /var/kerberos/krb5kdc/kpropd.acl:

host/master-kdc.customer.com@CUSTOMER.HDP
host/slave-kdc.customer.com@CUSTOMER.HDP
Now start the KDC and kadmin processes on the Master KDC only:

shell% systemctl enable krb5kdc 
shell% systemctl start krb5kdc 
shell% systemctl enable kadmin 
shell% systemctl start kadmin  
The KDC database is then initialized with the following command, executed from the Master KDC:

shell% kdb5_util create -s 
Loading random data 
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'CUSTOMER.HDP', 
master key name 'K/M@CUSTOMER.HDP' 
You will be prompted for the database Master Password. 
It is important that you NOT FORGET this password. 
Enter KDC database master key: <db_password>
Re-enter KDC database master key to verify: <db_password>
An administrator must be created to manage the Kerberos realm. The following command is used to create the administration principal from the Master KDC:

shell% kadmin.local -q "addprinc admin/admin" 
Authenticating as principal root/admin@CUSTOMER.HDP with password. 
WARNING: no policy specified for admin/admin@CUSTOMER.HDP; defaulting to no policy 
Enter password for principal "admin/admin@CUSTOMER.HDP": <admin_password>
Re-enter password for principal "admin/admin@CUSTOMER.HDP": <admin_password>
Principal "admin/admin@CUSTOMER.HDP" created. 
Host keytabs must now be created for the SLAVE KDC. Execute the following commands from the Master KDC:

shell% kadmin
kadmin: addprinc -randkey host/master-kdc.customer.com
kadmin: addprinc -randkey host/slave-kdc.customer.com
Extract the host key for the Slave KDC and store it on the hosts keytab file, /etc/krb5.keytab.slave:

kadmin: ktadd –k /etc/krb5.keytab.slave host/slave-kdc.customer.com
Copy /etc/krb5.keytab.slave to slave-kdc.customer.com and rename the file to /etc/krb5.keytab

Update /etc/services on each KDC host, if not present:

krb5_prop       754/tcp               # Kerberos slave propagation
Install xinetd on the hosts of the Master and Slave KDC, if not already installed, to enable kpropd to execute:

yum install xinetd
Create the configuration for kpropd on both the Master and Slave KDC hosts:

Create /etc/xinetd.d/krb5_prop with the following contents.

Create /etc/xinetd.d/krb5_prop with the following contents.
service krb_prop
{
        disable         = no
        socket_type     = stream
        protocol        = tcp
        user            = root
        wait            = no
        server          = /usr/sbin/kpropd
}
Configure xinetd to run as a persistent service on both the Master and Slave KDC hosts:

systemctl enable xinetd.service
systemctl start xinetd.service
Copy the following files from the Master KDC host to the Slave KDC host:

/etc/krb5.conf 
/var/kerberos/krb5kdc/kadm5.acl 
/var/kerberos/krb5kdc/kdc.conf
/var/kerberos/krb5kdc/kpropd.acl
/var/kerberos/krb5kdc/.k5.CUSTOMER.HDP
Perform the initial KDC database propagation to the Slave KDC:

shell% kdb5_util dump /usr/local/var/krb5kdc/slave_datatrans
shell% kprop -f /usr/local/var/krb5kdc/slave_datatrans slave-kdc.customer.com
The Slave KDC may be started at this time:

shell% systemctl enable krb5kdc 
shell% systemctl start krb5kdc 
Script to propagate the updates from the Master KDC to the Slave KDC. Create a cron job, or the like, to run this script on a frequent basis.

#!/bin/sh
#/var/kerberos/kdc-slave-propogate.sh
kdclist = "slave-kdc.customer.com"
/sbin/kdb5_util dump /usr/local/var/krb5kdc/slave_datatrans
for kdc in $kdclist
do
    /sbin/kprop -f /usr/local/var/krb5kdc/slave_datatrans $kdc
done




Ref:[https://community.hortonworks.com/articles/92333/configure-two-kerberos-kdcs-as-a-masterslave.html](https://community.hortonworks.com/articles/92333/configure-two-kerberos-kdcs-as-a-masterslave.html)
