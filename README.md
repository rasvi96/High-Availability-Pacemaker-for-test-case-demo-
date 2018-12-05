# Pacemaker-for-test-case
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
1. Create two Ubuntu 14.04 droplets( for demo you can create it as two VMs in Oracle VirtualBox or VMware Workstation)
2. Installing Pacemaker and Corosync
3. Configure Corosync(Adding Floating IP)
4. Start and Configure Pacemaker
5. Configure Cluster Properties
6. Testing High Availability Setup
