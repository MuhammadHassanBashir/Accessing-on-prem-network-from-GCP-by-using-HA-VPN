# Accessing on prem network from GCP by using HA VPN

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
      Project id:world-learning-400909              Project ID for the existing GCP project where the environment will be set up.
      Region: us-central1                           Deployment region for the GCP environment
      VPC CIDR: 10.0.1.0/24                         GCP network CIDR subnet range
      VPC Name: vpc                                 Name for the GCP VPC

HA-VPN CONFIGURATION
--------------------

## Accessing the GCP project on local machine or vm

      gcloud config set project **YOU-PROJECT-ID**

## Create a VPC named "vpc"
### Enable `compute.googleapis.com` if prompted to do so.

      gcloud compute networks create vpc --bgp-routing-mode=global \
        --subnet-mode=custom

## Create a firewall rule enabling ICMP and SSH ingress

      gcloud compute firewall-rules create allow-icmp-ssh --network vpc \
        --allow tcp:22,icmp

## Create a subnet 

      gcloud compute networks subnets create ew1-subnet --network=vpc \
        --range=10.0.1.0/24 --region=us-central1

## Create the VPN gateway which will terminate the ipsec tunnels
      
      gcloud compute vpn-gateways create gcp-to-onprem \
        --network=vpc \
        --region=us-central1
      
      It will be giving you tunnel **interface0 and interface1** ips that we will be using in our on prem section in **ipsec.conf** file.

## Create the external vpn gateway object
      
      gcloud compute external-vpn-gateways create onprem-vpn-gateway \
        --interfaces 0=34.35.43.205    

      Here you need to give on-prem public ip address. In our case we had configured our on-prem vm in gcp cloud under different region named africa-south1, So you need to assign the Public IP address of privioned vm here.

      Here you are basically peering vpn gateway with destination ip Address of on-prem vm..

## Create a Cloud Router to terminate the BGP sessions
     
      gcloud compute routers create vpn-router \
        --region=us-central1 \
        --network=vpc \
        --asn=64512 
      
## Announce Cloud DNS ranges
      
      gcloud compute routers update vpn-router \
         --advertisement-mode custom \
         --region=us-central1 \
         --set-advertisement-groups all_subnets \
         --set-advertisement-ranges 10.0.1.0/24       
      
      Advertising all subnet or specfic subnet(10.0.1.0/24) ranges through vpn gateway using cloud router..

## Create the VPN tunnels (tunnel-0)
      
      gcloud compute vpn-tunnels create tunnel-0 \
        --peer-external-gateway=onprem-vpn-gateway \
        --peer-external-gateway-interface=0  \
        --region=us-central1 \
        --ike-version=2 \
        --shared-secret=changeme-ike-secret \
        --router=vpn-router \
        --vpn-gateway=gcp-to-onprem \
        --interface=0

## Create the VPN tunnels (tunnel-1)
      
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

## Create the interface for the Cloud Router (0)
      
      gcloud compute routers add-interface vpn-router \
        --interface-name=vpn-if-0 --vpn-tunnel=tunnel-0 \
        --ip-address=169.254.1.2 --mask-length=30 --region=us-central1

## Create the BGP peer object for the Cloud Router interface (0)

      gcloud compute routers add-bgp-peer vpn-router \
        --peer-name=bgp-tunnel-0 \
        --peer-asn=65534 \
        --interface=vpn-if-0 \
        --peer-ip-address=169.254.1.1 \
        --region=us-central1

## Create the interface for the Cloud Router (1)

      gcloud compute routers add-interface vpn-router \
        --interface-name=vpn-if-1 --vpn-tunnel=tunnel-1 \
        --ip-address=169.254.1.6 --mask-length=30 --region=us-central1

## Create the BGP peer object for the Cloud Router interface (1)
      
      gcloud compute routers add-bgp-peer vpn-router \
        --peer-name=bgp-tunnel-1 \
        --peer-asn=65534 \
        --interface=vpn-if-1 \
        --peer-ip-address=169.254.1.5 \
        --region=us-central1

      It will get 169.254.. IP range from bgp global network. Use as it is...

## Creating test VM on subnet

      gcloud compute instances create test-ha-vpn --zone=us-central1-c --machine-type=f1-micro --subnet=ew1-subnet --create-disk=image=projects/debian-cloud/global/images/debian-10-buster-v20220118,mode=rw,size=10,type=projects/world-learning-400909/zones/europe-west1-b/diskTypes/pd-balanced --metadata=startup-script='#! /bin/bash
  apt update
  apt -y install apache2 iproute2 iputils-ping dnsutils'


ON-PREM SIDE
---------------
      
      Name        Value                                   Description
      External IP:34.35.43.205                            External IP address
      Internal IP:10.0.0.2/24                             Internal IP address, which will be configured on a loopback interface
      On premises CIDR,10.0.0.0/24,                       Onprem network CIDR
      On premises Private DNS Zone:onprem.example.com     On premises Private DNS Zone


# Configuration of the on-premises environment

The following commands install the required packages and binary files, create the required configuration files and start StrongSwan, FRR, CoreDNS and Nginx. Make sure to adapt them according to your existing environment.

## Install required packages
      
      apt update
      apt install nginx strongswan libstrongswan-standard-plugins \ 
        frr iproute2 -y

## Configure a simulated private network on loopback

      ip addr add 10.0.0.2/24 dev lo

# Firewall configuration
      sudo ufw allow from 10.0.0.0/8           on-prem network 
      sudo ufw allow from 169.254.0.0/16       bgp range
      sudo ufw allow from 172.16.0.0/12
      sudo ufw allow from 34.35.43.205/16      on-prem public ip      
      sudo ufw allow from 10.0.1.0/24   -----> Allowing gcp subnet range 
      sudo ufw allow 500/udp
      sudo ufw allow ssh
      sudo ufw default deny incoming
      sudo ufw --force enable
      
# Setup nginx 

      rm /var/www/html/* && echo "Greetings from $(curl -s ifconfig.me)!" >>/var/www/html/index.html

# Download CoreDNS
      
      curl -s -L https://github.com/coredns/coredns/releases/download/v1.8.7/coredns_1.8.7_linux_amd64.tgz | tar xz -C /usr/bin
      
# Setup IPSec secrets
      
      cat <<'EOF' >/etc/ipsec.secrets
       : PSK "changeme-ike-secret"
      EOF

# Setup IPSec

      cat <<'EOF' >/etc/ipsec.conf
      conn %default
        ikelifetime=600m
        keylife=180m
        rekeymargin=3m
        keyingtries=3
        keyexchange=ikev2
        mobike=no
        ike=aes256gcm16-sha512-modp2048
        esp=aes256gcm16-sha512-modp8192
        authby=psk
        left=%any
        leftid=%any
        leftsubnet=0.0.0.0/0
        leftauth=psk
        rightauth=psk
        rightsubnet=0.0.0.0/0
        type=tunnel
        auto=start
        dpdaction=restart
        closeaction=restart
        mark=%unique
      conn gcp
        leftupdown="/var/lib/strongswan/ipsec-vti.sh 0 169.254.1.2/30 169.254.1.1/30"
        right=34.157.101.116          #ipaddress of vpn gateway interface0
        rightid=34.157.101.116
      conn gcp2
        leftupdown="/var/lib/strongswan/ipsec-vti.sh 1 169.254.1.6/30 169.254.1.5/30"
        left=%any
        right=35.220.91.217   #ipaddress of vpn gateway interface1
        rightid=35.220.91.217
      EOF

# Setup IPSec VTI script

# BEGIN

      cat <<'EOF' >/var/lib/strongswan/ipsec-vti.sh
      #!/bin/bash
      # Copyright 2022 Google LLC
      #
      # Licensed under the Apache License, Version 2.0 (the "License");
      # you may not use this file except in compliance with the License.
      # You may obtain a copy of the License at
      #
      #      https://www.apache.org/licenses/LICENSE-2.0
      #
      # Unless required by applicable law or agreed to in writing, software
      # distributed under the License is distributed on an "AS IS" BASIS,
      # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
      # See the License for the specific language governing permissions and
      # limitations under the License.
      
      # originally published at
      # https://cloud.google.com/community/tutorials/using-cloud-vpn-with-strongswan
      
      set -o nounset
      set -o errexit
      
      IP=$(which ip)
      
      PLUTO_MARK_OUT_ARR=(${PLUTO_MARK_OUT//// })
      PLUTO_MARK_IN_ARR=(${PLUTO_MARK_IN//// })
      
      VTI_TUNNEL_ID=${1}
      VTI_REMOTE=${2}
      VTI_LOCAL=${3}
      
      LOCAL_IF="${PLUTO_INTERFACE}"
      VTI_IF="vti${VTI_TUNNEL_ID}"
      # GCP's MTU is 1460
      GCP_MTU="1460"
      # ipsec overhead is 73 bytes, we need to compute new mtu.
      VTI_MTU=$((GCP_MTU-73))
      
      case "${PLUTO_VERB}" in
          up-client)
              sudo ${IP} link add ${VTI_IF} type vti local ${PLUTO_ME} remote ${PLUTO_PEER} okey ${PLUTO_MARK_OUT_ARR[0]} ikey ${PLUTO_MARK_IN_ARR[0]}
              sudo ${IP} addr add ${VTI_LOCAL} remote ${VTI_REMOTE} dev "${VTI_IF}"
              sudo ${IP} link set ${VTI_IF} up mtu ${VTI_MTU}
      
              # Disable IPSEC Policy
              sudo /sbin/sysctl -w net.ipv4.conf.${VTI_IF}.disable_policy=1
      
              # Enable loosy source validation, if possible. Otherwise disable validation.
              sudo /sbin/sysctl -w net.ipv4.conf.${VTI_IF}.rp_filter=2 || sysctl -w net.ipv4.conf.${VTI_IF}.rp_filter=0
      
              # If you would like to use VTI for policy-based you shoud take care of routing by yourselv, e.x.
              if [[ "${PLUTO_PEER_CLIENT}" != "0.0.0.0/0" ]]; then
                  ${IP} r add "${PLUTO_PEER_CLIENT}" dev "${VTI_IF}"
              fi
              ;;
          down-client)
              sudo ${IP} tunnel del "${VTI_IF}"
              ;;
      esac
      
      # Enable IPv4 forwarding
      sudo /sbin/sysctl -w net.ipv4.ip_forward=1
      
      # Disable IPSEC Encryption on local net
      sudo /sbin/sysctl -w net.ipv4.conf.${LOCAL_IF}.disable_xfrm=1
      sudo /sbin/sysctl -w net.ipv4.conf.${LOCAL_IF}.disable_policy=1
      EOF

      #END 
      
      Use file as it is...

# Setup BGP routing
      
      cat <<'EOF' >/etc/frr/frr.conf
      frr defaults traditional
      hostname vpngw
      log syslog informational
      no ipv6 forwarding
      service integrated-vtysh-config
      !
      router bgp 65534
      neighbor 169.254.1.2 remote-as 64512
      neighbor 169.254.1.6 remote-as 64512
      !
      address-family ipv4 unicast
        redistribute connected
        neighbor 169.254.1.2 route-map gcp in
        neighbor 169.254.1.2 route-map gcp out
        neighbor 169.254.1.6 route-map gcp in
        neighbor 169.254.1.6 route-map gcp out
      exit-address-family
      !
      route-map gcp permit 10
      !
      line vty
      !
      EOF
                  
      Use file as it is

# Setup BGP daemon

      cat <<'EOF' >/etc/frr/daemons
      bgpd=yes
      ospfd=no
      ospf6d=no
      ripd=no
      ripngd=no
      isisd=no
      pimd=no
      ldpd=no
      nhrpd=no
      eigrpd=no
      babeld=no
      sharpd=no
      pbrd=no
      bfdd=no
      fabricd=no
      vrrpd=no
      vtysh_enable=yes
      zebra_options="  -A 127.0.0.1 -s 90000000"
      bgpd_options="   -A 127.0.0.1"
      EOF

# Setup CoreDNS
      
      cat <<'EOF' >/lib/systemd/system/coredns.service
      [Unit]
      Description=CoreDNS DNS server
      Documentation=https://coredns.io
      After=network.target
      [Service]
      PermissionsStartOnly=true
      LimitNOFILE=1048576
      LimitNPROC=512
      CapabilityBoundingSet=CAP_NET_BIND_SERVICE
      AmbientCapabilities=CAP_NET_BIND_SERVICE
      NoNewPrivileges=true
      User=root
      WorkingDirectory=~
      ExecStart=/usr/bin/coredns -conf=/etc/coredns/Corefile
      ExecReload=/bin/kill -SIGUSR1 $MAINPID
      Restart=on-failure
      [Install]
      WantedBy=multi-user.target
      EOF

# Create CoreDNS configuration
      
      mkdir /etc/coredns
      cat <<'EOF' >/etc/coredns/Corefile
      . {
          hosts /etc/coredns/onprem-hosts onprem.example.com {
              127.0.0.1   localhost.onprem.example.com localhost
          }
          reload
          log
          errors
      }
      EOF

# Create CoreDNS onprem-hosts file
      
      cat <<'EOF' >/etc/coredns/onprem-hosts
      127.0.0.10 ten
      EOF

# Finishing touches
      
      chmod +x /var/lib/strongswan/ipsec-vti.sh
      systemctl enable coredns.service
      service coredns restart
      service strongswan-starter restart
      service frr restart      

## SITE LINK

    https://medium.com/@sruffilli/setting-up-a-simulated-on-prem-environment-for-gcp-90dcbb2d57f8
