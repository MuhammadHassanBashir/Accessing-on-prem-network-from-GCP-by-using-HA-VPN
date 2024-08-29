# Accessing-on-prem-network-from-GCP-by-using-HA-VPN
Setting up a simulated on-prem environment for GCP

This guide is meant to setup a basic simulated on-prem environment, which configures IPSec (strongSwan), BGP (frr) and DNS (CoreDNS).

On the GCP side, an highly available VPN, routing and DNS setup is configured leveraging Google Cloud VPN HA and Google Cloud DNS.

A step-by-step procedure is provided outlining all the commands and configurations required to spin-up the required infrastructure on GCP and to configure your “on-prem” laboratory.

While this setup is fully tested at the time of writing and should work off-the-shelf, basic familiarity with the gcloud CLI, Linux administration and network protocols is recommended and will be useful in case something goes south.



      Name,Value,Description
      Cloud Private DNS Zone:gcp.example.com        On premises Private DNS Zone
      Cloud VPN Gateway INTERFACE0:34.157.101.116  "IP for the VPN Gateway INTERFACE0, generated when creating the VPN Gateway"
      Cloud VPN Gateway INTERFACE1:35.220.91.217   "IP for the VPN Gateway INTERFACE1, generated when creating the VPN Gateway"
      Project id:                                   Project ID for the existing GCP project where the environment will be set up.
      Region: us-central1                           Deployment region for the GCP environment
      VPC CIDR: 10.0.1.0/24                         GCP network CIDR subnet range
      VPC Name: vpc                                 Name for the GCP VPC












## SITE LINK

    https://medium.com/@sruffilli/setting-up-a-simulated-on-prem-environment-for-gcp-90dcbb2d57f8
