# CrowdSecurity Blocklist for Digital Ocean

Simple script to download the list of CIDR that [Digital Ocean](https://www.digitalocean.com/) publishes ([here](https://digitalocean.com/geo/google.csv)) and convert them to a CSV list to re-import using the `cscli decisions import` command in order to block all traffic from Digital Ocean.  

NOTE : My experience is the list is not quite complete.  It contains a massive range of CIDR, and blocked DO traffic on my servers has dropped considerably, but I do still see the occasional alert for an IP that does map back to DO.  Not sure how often its updated - I can't believe it's really in their interest to do it too often as my internet searching whilst looking for a pre-existing blocklist suggests that almost everyone who's looking for an exhaustive list of DO CIDRs is doing it for the same reason I am :D 

## Why bother?

- There is no valid use case for Digital Ocean traffic to access services I publish
- it is an absolute cesspit of all sorts of malicious traffic.
- They seem entirely uninterested in making any sort of sustainable change to prevent/reduce the number of bad actors using their service(s)

I had crowdsec protecting individual services with specific scenarios, but the amount of IPs that were from DO meant that I'd just save myself the bother and block them all.

## How does it work
It's fairly self explanatory, and will automatically download, enrich and then apply the blocklist.  At this stage, you'll need to change variables in the script 

### TODO
- Allow command line options  :
  - download,
  - download and apply
  - download, delete and apply
  - pass values to override the default script values

- Support other providers with similar lists...
