# Accessing-on-prem-network-from-GCP-by-using-HA-VPN
Setting up a simulated on-prem environment for GCP

This guide is meant to setup a basic simulated on-prem environment, which configures IPSec (strongSwan), BGP (frr) and DNS (CoreDNS).

On the GCP side, an highly available VPN, routing and DNS setup is configured leveraging Google Cloud VPN HA and Google Cloud DNS.

A step-by-step procedure is provided outlining all the commands and configurations required to spin-up the required infrastructure on GCP and to configure your “on-prem” laboratory.

While this setup is fully tested at the time of writing and should work off-the-shelf, basic familiarity with the gcloud CLI, Linux administration and network protocols is recommended and will be useful in case something goes south.


## GCP environment

      Name,Value,Description
      Cloud Private DNS Zone:gcp.example.com        On premises Private DNS Zone
      Cloud VPN Gateway INTERFACE0:34.157.101.116  "IP for the VPN Gateway INTERFACE0, generated when creating the VPN Gateway"
      Cloud VPN Gateway INTERFACE1:35.220.91.217   "IP for the VPN Gateway INTERFACE1, generated when creating the VPN Gateway"
      Project id:                                   Project ID for the existing GCP project where the environment will be set up.
      Region: us-central1                           Deployment region for the GCP environment
      VPC CIDR: 10.0.1.0/24                         GCP network CIDR subnet range
      VPC Name: vpc                                 Name for the GCP VPC

HA-VPN CONFIGURATION
--------------------

# Accessing the GCP project on local machine or vm

      gcloud config set project **YOU-PROJECT-ID**

# Create a VPC named "vpc"
# Enable `compute.googleapis.com` if prompted to do so.

      gcloud compute networks create vpc --bgp-routing-mode=global \
        --subnet-mode=custom

# Create a firewall rule enabling ICMP and SSH ingress

      gcloud compute firewall-rules create allow-icmp-ssh --network vpc \
        --allow tcp:22,icmp

# Create a subnet 

      gcloud compute networks subnets create ew1-subnet --network=vpc \
        --range=10.0.1.0/24 --region=us-central1

# Create the VPN gateway which will terminate the ipsec tunnels
      
      gcloud compute vpn-gateways create gcp-to-onprem \
        --network=vpc \
        --region=us-central1
      
      It will be giving you tunnel interface0 and interface1 ips that we will be using in our on prem section in **ipsec.conf** file.

# Create the external vpn gateway object
      
      gcloud compute external-vpn-gateways create onprem-vpn-gateway \
        --interfaces 0=34.35.43.205    

      Here you need to give on-prem public ip address. In our case we had configured our on-prem vm in gcp cloud under different region named africa-south1, So you need to assign the Public IP address of privioned vm here.

      Here you are basically peering vpn gateway with destination ip Address of on-prem vm..

# Create a Cloud Router to terminate the BGP sessions
     
      gcloud compute routers create vpn-router \
        --region=us-central1 \
        --network=vpc \
        --asn=64512 
      
# Announce Cloud DNS ranges
      
      gcloud compute routers update vpn-router \
         --advertisement-mode custom \
         --region=us-central1 \
         --set-advertisement-groups all_subnets \
         --set-advertisement-ranges 10.0.1.0/24       
      
      Advertising all subnet or specfic subnet(10.0.1.0/24) ranges through vpn gateway using cloud router..

# Create the VPN tunnels (tunnel-0)
      
      gcloud compute vpn-tunnels create tunnel-0 \
        --peer-external-gateway=onprem-vpn-gateway \
        --peer-external-gateway-interface=0  \
        --region=us-central1 \
        --ike-version=2 \
        --shared-secret=changeme-ike-secret \
        --router=vpn-router \
        --vpn-gateway=gcp-to-onprem \
        --interface=0

# Create the VPN tunnels (tunnel-1)
      
      gcloud compute vpn-tunnels create tunnel-1 \
        --peer-external-gateway=onprem-vpn-gateway \
        --peer-external-gateway-interface=0 \
        --region=us-central1 \
        --ike-version=2 \
        --shared-secret=changeme-ike-secret \
        --router=vpn-router \
        --vpn-gateway=gcp-to-onprem \
        --interface=1

      You can manage pre-shared key accordingly.. In our case we have used as it is...

GCP-SIDE
-----------

      Name                   Value                  Description
      Cloud Private DNS Zone:gcp.example.com        On premises Private DNS Zone
      Cloud VPN Gateway INTERFACE0:34.157.101.116  "IP for the VPN Gateway INTERFACE0, generated when creating the VPN Gateway"
      Cloud VPN Gateway INTERFACE1:35.220.91.217   "IP for the VPN Gateway INTERFACE1, generated when creating the VPN Gateway"
      Project id:                                   Project ID for the existing GCP project where the environment will be set up.
      Region: us-central1                           Deployment region for the GCP environment
      VPC CIDR: 10.0.1.0/24                         GCP network CIDR subnet range
      VPC Name: vpc                                 Name for the GCP VPC

ON-PREM SIDE
---------------
      
      Name        Value                                   Description
      External IP:34.35.43.205                            External IP address
      Internal IP:10.0.0.2/24                             Internal IP address, which will be configured on a loopback interface
      On premises CIDR,10.0.0.0/24,                       Onprem network CIDR
      On premises Private DNS Zone:onprem.example.com     On premises Private DNS Zone



## SITE LINK

    https://medium.com/@sruffilli/setting-up-a-simulated-on-prem-environment-for-gcp-90dcbb2d57f8
