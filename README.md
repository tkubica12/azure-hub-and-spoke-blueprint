# Azure Blueprint for hub-and-spoke environment
This repo contains example Blueprint for hub-and-spoke topology with enterprise governance. Each spoke subscription, eg. business unit project, must fullfil following requirements: 

- Project lead must be subscription Owner so he can manage access to members of his team and service principals
- Azure Policies must be in place to enforce IT requirements such as:
  - regional restrictions (eg. only West Europe where Express Route is build)
  - enforce audit logging on services
  - enforce tagging structure
  - limit IaaS VMs to use only IT provided hardened images
  - forbid creation of certain resources such as public IP, Azure VPN, new VNETs
- RBAC rule to assign project lead as Owner
- Basic networking setup needs to be deployed (VNET, subnets, forced routing to firewall in hub subscription)
- Project lead (Owner) must not be able to remove policies
- Project lead (Owner) must not be able to modify networking setup