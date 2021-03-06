# FreeIPA-Cluster #

This is a very small guide to deploy FreeIPA in HA, included settings for clients

This guide is written for CentOS/RHEL but, apart installation steps, you can adopt in other system, after FreeIPA installation

* Step 1 - Update system<br/>
`yum -y update`

* Step 2 - Modify hostname<br/>
`hostnamectl set-hostname ipa.mydomain.local`

* Step 3 - Modify reference on /etc/hosts<br/>
`echo "192.168.1.10 ipa.mydomain.local ipa" >> /etc/hosts`

* Step 4 - ONLY FOR RHEL - Enable subscription<br/>
`subscription-manager repos --enable rhel-7-server-optional-rpms`

* Step 5 - Packages installation<br/>
`yum -y install ipa-server ipa-server-dns bind bind-dyndb-ldap`

* Step 6 - Start and enable named<br/>
`systemctl enable named && systemctl start named`

* Step 7 - Modify DNS reference<br/>
`nmcli con mod eth0 ipv4.dns 127.0.0.1 && nmcli con up eth0`

* Step 8 - Setup FreeIPA server<br/>
`ipa-server-install`

	  Answer:
	  Do you want to configure integrated DNS (BIND)? [no]: yes
	  Server host name [ipa.mydomain.local]:
	  Please confirm the domain name [domain.local]
	  Please provide a realm name [MYDOMAIN.LOCAL]:
	  Directory Manager password:
	  Password (confirm):
	  IPA admin password:
	  Password (confirm):
	  Do you want to configure DNS forwarders? [yes]:
	  (If you have or want a forwarder, otherwise answer NO)

	  In case of YES:
       Following DNS servers are configured in /etc/resolv.conf: 127.0.0.1
       Do you want to configure these servers as DNS forwarders? [yes]: >> NO
       Enter an IP address for a DNS forwarder, or press Enter to skip: 8.8.8.8
       DNS forwarder 8.8.8.8 added. You may add another.
       Enter an IP address for a DNS forwarder, or press Enter to skip: 8.8.4.4
	   
	   In case of NO:
       Do you want to configure DNS forwarders? [yes]: no
	   No DNS forwarders configured
	   
	   Do you want to search for missing reverse zones? [yes]:
	   Do you want to create reverse zone for IP 192.168.100.10 [yes]:
	   Please specify the reverse zone name [100.168.192.in-addr.arpa.]:
	   
	   Continue to configure the system with these values? [no]: >> YES
	   ===================================================
	   Setup complete
	 
	  Next steps:
	  1. You must make sure these network ports are open:
       
	   TCP Ports:
         * 80, 443: HTTP/HTTPS
		 * 389, 636: LDAP/LDAPS
		 * 88, 464: kerberos
		 * 53: bind

	   UDP Ports:
	     * 88, 464: kerberos
		 * 53: bind
		 * 123: ntp
		2. You can now obtain a kerberos ticket using the command: 'kinit admin'
      This ticket will allow you to use the IPA tools (e.g., ipa user-add)
      and the web user interface.
	  
	  Be sure to back up the CA certificates stored in /root/cacert.p12
	  These files are required to create replicas. The password for thes files is the Directory Manager password

* Step 8 - Firewall<br/>
  `firewall-cmd --permanent --add-service={ntp,http,https,ldap,ldaps,kerberos,kpasswd}`<br/>
  `firewall-cmd --permanent --add-port=53/udp`<br/>
  `firewall-cmd --reload`

* Step 9 - Check if ipa work<br/>
  On the ipa server cli<br/>
  `kinit admin`
    
    Password for admin@MYDOMAIN.LOCAL:
	
	`ipa user-find`
      
      
      --------------
      1 user matched
      --------------
      User login: admin
      Last name: Administrator
      Home directory: /home/admin
      Login shell: /bin/bash
      Principal alias: admin@MYDOMAIN.LOCAL
      UID: 1608400000
      GID: 1608400000
      Account disabled: False
      ----------------------------
      Number of entries returned 1
      ----------------------------

# Additional Steps for Slave replication
* Step 11 - Slave installation (IP: 192.168.100.11 - hostname: ipaslave.mydomain.local)
  On the slave:
    `yum -y update`<br/>
    `hostnamectl set-hostname ipa.mydomain.local`<br/>
    `echo "192.168.100.11 ipa-slave.mydomain.local ipa-slave" >> /etc/hosts`<br/>
    `nmcli con mod eth0 ipv4.dns 192.168.100.10 && nmcli con up eth0`<br/>
	(DNS IP address specified MUST is a MASTER IPA)<br/>
    `yum -y install ipa-server ipa-server-dns bind bind-dyndb-ldap`<br/>
    `systemctl enable named && systemctl start named`<br/>
  
  On the master ipa:<br/>
    `ipa dnsrecord-add domain.local ipa-slave --a-rec 192.168.100.11`<br/>
    `ipa-replica-prepare ipa-slave.mydomain.local --ip-address 192.168.100.11`<br/>
  
  Directory Manager (existing master) password:<br/>
     `scp /var/lib/ipa/replica-info-ipa-slave.mydomain.local.gpg \ `<br/>
	 `root@ipa-slave.mydomain.local:/var/lib/ipa/`<br/>
	
    `firewall-cmd --add-service=freeipa-replication --permanent`<br/>
	
    `firewall-cmd --reload`<br/>
  
  On the slave:<br/>
    `dig -x 192.168.100.11` (check if ptr reord is ok)<br/>
    `firewall-cmd --add-service={ssh,dns,freeipa-ldap,freeipa-ldaps,freeipa-replication} --permanent`
	
    `firewall-cmd --reload`<br/>
	
    `ipa-replica-install --setup-ca --setup-dns --no-forwarders \`<br/>
	`/var/lib/ipa/replica-info-ipa-slave.mydomain.local.gpg`<br/>

      Directory Manager (existing master) password:
      Run connection check to master
      Check connection from replica to remote master 'ipa.mydomain.local':
      Directory Service: Unsecure port (389): OK
      Directory Service: Secure port (636): OK
      Kerberos KDC: TCP (88): OK
      Kerberos Kpasswd: TCP (464): OK
      HTTP Server: Unsecure port (80): OK
      HTTP Server: Secure port (443): OK
      PKI-CA: Directory Service port (7389): OK
      The following list of ports use UDP protocol and would need to be checked manually:
        Kerberos KDC: UDP (88): SKIPPED
        Kerberos Kpasswd: UDP (464): SKIPPED
      Connection from replica to master is OK.
      Start listening on required ports for remote master check
      Get credentials to log in to remote master
        admin@MYDOMAIN.LOCAL password:
      Execute check on remote master
      .....
      .....
      Global DNS configuration in LDAP server is empty
      You can use 'dnsconfig-mod' command to set global DNS options that would override settings in local named.conf files
      Restarting the web server

Final step: installation client<br/>
`yum install ipa-client ipa-admintools`<br/>

Verify this setting in /etc/sssd/sssd.conf<br/>
ipa_server = _srv_, ipa-slave.mydomain.local
