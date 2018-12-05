# High Availability Pacemaker(for test case)
How To Create a High Availability Setup with Corosync, Pacemaker, and Floating IPs on Ubuntu 14.04 in simple method (for practice)

# Introduction
This tutorial will demonstrate how you can use Corosync and Pacemaker with a Floating IP to create a high availability (HA) server infrastructure on your Ubuntu 14.04 without paying any money for getting Floating IP.
# Corosync and Pacemaker
Corosync is an open source program that provides cluster membership and messaging capabilities, often referred to as the messaging layer, to client servers.
Pacemaker is an open source cluster resource manager (CRM), a system that coordinates resources and services that are managed and made highly available by a cluster. 
In essence, Corosync enables servers to communicate as a cluster, while Pacemaker provides the ability to control how the cluster behaves.
# Goal
Our goal is to create High Availability setup with corosync, pacemaker and floating IP. When completed, the HA setup will consist of two Ubuntu 14.04 servers in an active/passive configuration. This will be accomplished by pointing a Floating IP, which is how your users will access your web service, to point to the primary (active) server unless a failure is detected. If a failure is detected to the primary server(active) then the secondary server immediately become active and providing web service without any interruption.
# Steps to follow
1. Create two Ubuntu 14.04 droplets( for demo you can create it as two VMs in Oracle VirtualBox or VMWare Workstation)
2. Installing Pacemaker and Corosync
3. Configure Corosync(Adding Floating IP)
4. Start and Configure Pacemaker
5. Configure Cluster Properties
6. Testing High Availability Setup
7. Conclusion
# Creating Droplets
Create two Ubuntu 14.04 Operating Systems as two servers in Oracle VirtualBox(or VMWare Workstation).
# Configure Time Synchronization
Whenever you have multiple servers communicating with each other, especially with clustering software, it is important to ensure their clocks are synchronized. We'll use NTP (Network Time Protocol) to synchronize our servers.
On both servers, use this command to open a time zone selector:
```
sudo dpkg-reconfigure tzdata
```
Select your desired time zone. For example we'll choose ```India/Kolkata```
Next, update apt-get:
```
sudo apt-get update
```
Then install the ntp package with this command;
```
sudo apt-get -y install ntp
```
Your server clocks should now be synchronized using NTP.
# Configure Firewall
Corosync uses UDP transport between ports 5404 and 5406. If you are running a firewall, ensure that communication on those ports are allowed between the servers.
For example, if you're using iptables, you could allow traffic on these ports and eth1 (the private network interface) with these commands:
```
sudo iptables -A INPUT  -i eth1 -p udp -m multiport --dports 5404,5405,5406 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT  -o eth1 -p udp -m multiport --sports 5404,5405,5406 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```
It is advisable to use firewall rules that are more restrictive than the provided example.
# Install Corosync and Pacemaker
On both servers, install Corosync and Pacemaker using apt-get:
```
sudo apt-get install pacemaker
```
Note that Corosync is installed as a dependency of the Pacemaker package.
Corosync and Pacemaker are now installed but they need to be configured before they will do anything useful.
# Configure Corosync
Corosync must be configured so that our servers can communicate as a cluster.
# Create Cluster Authorization Key
In order to allow nodes to join a cluster, Corosync requires that each node possesses an identical cluster authorization key.
On the primary server, install the haveged package:
```
primary$ sudo apt-get install haveged
```
This software package allows us to easily increase the amount of entropy on our server, which is required by the corosync-keygen script.
On the primary server, run the corosync-keygen script:
```
primary$ sudo corosync-keygen
```
This will generate a 128-byte cluster authorization key, and write it to /etc/corosync/authkey.
Now that we no longer need the haveged package, let's remove it from the primary server:
```
primary$ sudo apt-get remove --purge haveged
primary$ sudo apt-get clean
```
On the primary server, copy the authkey to the secondary server:
```
primary$ sudo scp /etc/corosync/authkey username@secondary_ip:/tmp
```
On the secondary server, move the authkey file to the proper location, and restrict its permissions to root:
```
secondary$ sudo mv /tmp/authkey /etc/corosync
secondary$ sudo chown root: /etc/corosync/authkey
secondary$ sudo chmod 400 /etc/corosync/authkey
```
Now both servers should have an identical authorization key in the /etc/corosync/authkey file.
# Configure Corosync Cluster
In order to get our desired cluster up and running, we must set up these
On both servers, open the corosync.conf file for editing in your favorite editor (we'll use vi editor):
```
sudo vi /etc/corosync/corosync.conf
```
Here is a Corosync configuration file that will allow your servers to communicate as a cluster. Be sure to replace the highlighted parts with the appropriate values. bindnetaddr should be set to the private IP address of the server you are currently working on. The two other highlighted items should be set to the indicated server's private IP address. With the exception of the bindnetaddr, the file should be identical on both servers.

Replace the contents of corosync.conf with this configuration, with the changes that are specific to your environment:
```
totem {
  version: 2
  cluster_name: lbcluster
  transport: udpu
  interface {
    ringnumber: 0
    bindnetaddr: $server_private_IP_address(Floating IP, for example '192.168.56.200' without single quotes)
    broadcast: yes
    mcastport: 5405
  }
}

quorum {
  provider: corosync_votequorum
  two_node: 1
}

nodelist {
  node {
    ring0_addr: $primary_private_IP_address(replace with first ubuntu private IP)
    name: primary
    nodeid: 1
  }
  node {
    ring0_addr: $secondary_private_IP_address(replace with second Ubuntu private IP)
    name: secondary
    nodeid: 2
  }
}

logging {
  to_logfile: yes
  logfile: /var/log/corosync/corosync.log
  to_syslog: yes
  timestamp: on
}
```
Save and exit.

Next, we need to configure Corosync to allow the Pacemaker service.

On both servers, create the pcmk file in the Corosync service directory with an editor. We'll use vi:

sudo vi /etc/corosync/service.d/pcmk
Then add the Pacemaker service:

service {
  name: pacemaker
  ver: 1
}
Save and exit. 
This will be included in the Corosync configuration, and allows Pacemaker to use Corosync to communicate with our servers.
By default, the Corosync service is disabled. On both servers, change that by editing /etc/default/corosync:
```
sudo vi /etc/default/corosync
```
Change the value of START to yes:
```
vi /etc/default/corosync
START=yes
```
Save and exit. Now we can start the Corosync service.
On both servers, start Corosync with this command:
```
sudo service corosync start
```
Once Corosync is running on both servers, they should be clustered together. We can verify this by running this command:
```
sudo corosync-cmapctl | grep members
```
The output should look something like this, which indicates that the primary (node 1) and secondary (node 2) have joined the cluster:
```
corosync-cmapctl output:
runtime.totem.pg.mrp.srp.members.1.config_version (u64) = 0
runtime.totem.pg.mrp.srp.members.1.ip (str) = r(0) ip(primary_private_IP_address)
runtime.totem.pg.mrp.srp.members.1.join_count (u32) = 1
runtime.totem.pg.mrp.srp.members.1.status (str) = joined
runtime.totem.pg.mrp.srp.members.2.config_version (u64) = 0
runtime.totem.pg.mrp.srp.members.2.ip (str) = r(0) ip(secondary_private_IP_address)
runtime.totem.pg.mrp.srp.members.2.join_count (u32) = 1
runtime.totem.pg.mrp.srp.members.2.status (str) = joined
```
Now that you have Corosync set up properly, let's move onto configuring Pacemaker.
# Start and Configure Pacemaker
Pacemaker, which depends on the messaging capabilities of Corosync, is now ready to be started and to have its basic properties configured.
# Enable and Start Pacemaker
The Pacemaker service requires Corosync to be running, so it is disabled by default.
On both servers, enable Pacemaker to start on system boot with this command:
```
sudo update-rc.d pacemaker defaults 20 01
```
With the prior command, we set Pacemaker's start priority to 20. It is important to specify a start priority that is higher than Corosync's (which is 19 by default), so that Pacemaker starts after Corosync.
Now let's start Pacemaker:
```
sudo service pacemaker start
```
To interact with Pacemaker, we will use the crm utility.
Check Pacemaker with crm:
```
sudo crm status
```
This should output something like this (if not, wait for 30 seconds, then run the command again):
```
crm status:
Last updated: Fri Oct 16 14:38:36 2015
Last change: Fri Oct 16 14:36:01 2015 via crmd on primary
Stack: corosync
Current DC: primary (1) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
0 Resources configured

Online: [ primary secondary ]
```
There are a few things to note about this output. First, Current DC (Designated Coordinator) should be set to either primary (1) or secondary (2). Second, there should be 2 Nodes configured and 0 Resources configured. Third, both nodes should be marked as online. If they are marked as offline, try waiting 30 seconds and check the status again to see if it corrects itself.

From this point on, you may want to run the interactive CRM monitor in another SSH window (connected to either cluster node). This will give you real-time updates of the status of each node, and where each resource is running:
```
sudo crm_mon
```
The output of this command looks identical to the output of crm status except it runs continuously. If you want to quit, press Ctrl-C.
# Configure Cluster Properties
Now we're ready to configure the basic properties of Pacemaker. Note that all Pacemaker (crm) commands can be run from either node server, as it automatically synchronizes all cluster-related changes across all member nodes.
For our desired setup, we want to disable STONITH—a mode that many clusters use to remove faulty nodes—because we are setting up a two-node cluster. To do so, run this command on either server:
```
sudo crm configure property stonith-enabled=false
```
We also want to disable quorum-related messages in the logs:
```
sudo crm configure property no-quorum-policy=ignore
```
Again, this setting only applies to 2-node clusters.

If you want to verify your Pacemaker configuration, run this command:
```
sudo crm configure show
```
This will display all of your active Pacemaker settings. Currently, this will only include two nodes, and the STONITH and quorum properties you just set.
# Add FloatIP Resource
With our FloatIP OCF resource agent installed, we can now configure our FloatIP resource.
On either server, create the FloatIP resource with this command (be sure to specify the two highlighted parameters with your own information):
```
crm configure primitive FAILOVER-ADDR ocf:heartbeat:OPaddr2 params ip="$Floating IP" nic="eth0" op monitor interval="10s" meta is-managed="true"
```
Note: 
1.Use the Floating IP what you mentioned in Corosync configuration file.
2. Mention the nic interface which has the private IP of primary and secondary servers( it may be eth0 or eth1 or something else)
If you check the status of your cluster 
```
sudo crm status or sudo crm_mon
```
By running either of above commands, you can see that the resource agent is added.
```
Last updated: Fri Oct 16 14:38:36 2018
Last change: Fri Oct 16 14:36:01 2018 via crmd on primary
Stack: corosync
Current DC: primary (1) - partition with quorum
Version: 1.1.10-42f2063
2 Nodes configured
1 Resources configured

Online: [ primary secondary ]
FAILOVER-ADDR (ocf::heartbeat:IPaddr2): Started primary
```
# Test High Availability Setup
It's important to test that our high availability setup works, so let's do that now.
Currently, the primary server is started i.e., the primary server is in active condition which is accessed by host OS through Floating IP and the secondary server is in passive condition.
To test our setup, in host OS(may be Windows) open the command promt and just ping the Floating IP that you mentioned in Corosync configuration file
```
ping -t 192.168.56.200(type your Floating IP)
```
Next, restart the primary server
```
primary$ sudo reboot
```
And simultaneously, monitor the cluster resource manager by running the following command in secondary server
```
secondary$ crm_mon
```
In secondary server you can see that changes ```Started primary``` will change to ```Started secondary``` and also you can able to see a minute in TTL in command prompt in Windows(Host OS).
If you followed these steps properly, then you can see this output ideally.
# Useful commands for troubleshooting
```
sudo crm configure edit
```
You can set a node to standby, online mode
```
sudo crm node standby NodeName 
sudo crm node online NodeName
```
You can edit a resource, which allows you to reconfigure it, with this command:
```
sudo crm configure edit ResourceName
```
You can delete a resource, which must be stopped before it is deleted, with these command:
```
sudo crm resource stop ResourceName
sudo crm configure delete ResourceName
```
Lastly, the crm command can be run by itself to access an interactive crm prompt:
```
crm
```
# Conclusion
Congratulations! You now have a basic HA server setup using Corosync, Pacemaker, and a Floating IP.

# References
https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04
 
To learn more about NTP, check out this 

https://www.digitalocean.com/community/tutorials/additional-recommended-steps-for-new-ubuntu-14-04-servers#configure-timezones-and-network-time-protocol-synchronization
