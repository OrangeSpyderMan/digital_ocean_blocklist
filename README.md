# CrowdSecurity Blocklist for Digital Ocean

Simple script to download the list of CIDR that [Digital Ocean](https://www.digitalocean.com/) publishes ([here](https://digitalocean.com/geo/google.csv)) and convert them to a CSV list to re-import using the `cscli decisions import` command in order to block all traffic from Digital Ocean.  

## Why bother?

There is no valid use case for Digital Ocean traffic to access services I publish and it is an absolute cesspit of all sorts of malicious traffic.   I had crowdsec protecting individual services with specific scenarios, but the amount of IPs that were from DO meant that I'd just save myself the bother and block them all.
