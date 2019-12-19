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

## Setting-up hub subscription
ARM template hubNetwork.json will setup simple example of hub VNET with subnets ready for firewall and other components. For purpose of this demo we are not going to deploy actual firewalls (purpose is Blueprint demo, not networking itself).

```powershell
Set-AzContext -Subscription mojesub
New-AzResourceGroup -Name hub-networking-rg -Location westeurope
New-AzResourceGroupDeployment -ResourceGroup hub-networking-rg -TemplateFile arm/hubNetwork.json
```

## Preparing ARM template for scope
We need to create ARM template for spoke vnet and setup peering to hub network. 

With that we can currently take two approaches to automation:
- We can use nested deployment to also provision VNET peering, but that requires identity with access to both subscriptions. This means we will need to use user-managed identity and take care of its RBAC ourselves meaning adding this identity to spoke subscription as Owner before Blueprint assignment and removing it after (for security reasons).
- We can use system-managed identity when Blueprint will automatically create identity, make it owner during provisioning and remove this RBAC after deployment automatically. Drawback is this identity does not have rights to our hub subscription so currently cannot automate deployment VNET peering.

In order to properly configure all networking pieces we will use user-managed identity.

## Defining Blueprint
Create new Blueprint

![](images/01.png)

![](images/02.png)

Define Blueprint name and choose its location.

![](images/03.png)

Make sure it is defined on Management Group level.

![](images/04.png)

Start adding Artifacts.

![](images/05.png)

First we will add RBAC to subscription level. We want to assign project lead as Owner so he can add other members of his team or Service Principal accounts. As each subscription will have different project lead, we will make this parameter to be decided during Blueprint assignment.

![](images/06.png)

Next we will assign policies. For demonstration purposes we will do just one - restriction to locations.

![](images/07.png)

Then we are going to create resource group for networking components. For simplicity we want the same name in every subscription and always use West Europe, therefore we will set those in Blueprint definition directly rather than as parameter during assignment.

![](images/08.png)

We now add artifacts into this resource group.

![](images/09.png)

Note we plan to lock resources in this resource group so no one including subscription owner can modify those. Nevertheless should networking team need to do some temporary manual change, they can unassign blueprint to test things out. Therefore we will add networking team (Security Group in AAD) as Network Contributor (but note they are still not able to make changes as resource is locked).

![](images/10.png)

We will add additional artifact - ARM template to deploy spoke networking components and configure peering from spoke to hub and back. Note template is using nested deployment so we can configure peering from both sides. Since we need access to both spoke and hub subscription, we need to use user-managed identity for assignment (as we will see later). Copy contents of file hubNetworkWithPeering.json to GUI. Note there are parameters in ARM template and we will make them all decided during blueprint assignment phase.

![](images/11.png)

We now have Blueprint definition ready so let's save it.

![](images/13.png)

We will now publish our Blueprint.

![](images/14.png)

Publish as v1.

![](images/15.png)

## Prepare user-managed identity
We will create user-managed identity in hub subscription and give it Network Contributor role to hub-networking-rg.

First make sure you have PowerShell Az module (tested with version 3.2.0) and Az.ManagedIdentity module installed.
```powershell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
Install-Module -Name Az.ManagedServiceIdentity -AllowClobber -Scope CurrentUser
```

Create managed identity in hub-networking-rg and assign Network Contributor rights for this identity on scope of hub-networking-rg

```powershell
Set-AzContext -Subscription mojesub
New-AzUserAssignedIdentity -ResourceGroupName hub-networking-rg -Name blueprintRobot

New-AzRoleAssignment -ObjectId (Get-AzUserAssignedIdentity -Name blueprintRobot -ResourceGroupName hub-networking-rg).PrincipalId `
    -RoleDefinitionName "Network Contributor" `
    -ResourceGroupName "hub-networking-rg"
```

Store blueprintRobot GUID for later use.
```powershell
(Get-AzUserAssignedIdentity -Name blueprintRobot -ResourceGroupName hub-networking-rg).PrincipalId
```

## Assign and test behavior
As we need to use user-managed identity we first need to temporarily give blueprintRobot identity Owner role on spoke subscription. Make sure you use ObjectId of blueprintRobot identity.

```powershell
Set-AzContext -Subscription mojesub2
New-AzRoleAssignment -ObjectId 61895eee-e28b-428f-9b66-0cf0e6f5ce45 -RoleDefinitionName "Owner"
```

Assign Blueprint.

![](images/16.png)

Choose your spoke subscription.

![](images/17.png)

We want to lock all deployed resources and policies so no one even Owner of subscription can modify those. We will use user-assigned identity for deployment and lock.

![](images/18.png)

Let's assign parameters. We will insert project lead account who will be Owner of subscription and specify allowed regions (West Europe in our case).

![](images/19.png)

Specify project name prefix address ranges allocated to this spoke network. Then we are done so click Assign.

![](images/20.png)

You can track progress of blueprint deployment.

![](images/21.png)

Let's check what has been configured in spoke subscription. We will see resource group with networking created.

![](images/22.png)

We can check VNET peering is configured and in connected state.

![](images/23.png)

Note networkTeam has beed added to networking-rg, but since resources are locked, they cannot modify settings. Nevertheless they are able to check configurations and do troubleshooting. Should they need to modify settings we need to modify Blueprint, create new version and update assignment. When temporary changes are needed eg. for troubleshooting purposes we can unassign Blueprint and networkTeam can do changes for testing. After correct setup is found, we would create new version of blueprint and assign it back to spoke subscription.

![](images/24.png)

Let's have a look on Deny Assignments. 

![](images/25.png)

Note resource is locked so nobody including Owner of subscription can modify resources in this resource group except for blueprintRobot. Later on we will remove blueprintRobot as subscription Owner so even this account cannot do any modifications.

![](images/26.png)

Azure Policy has also been assigned to our subscription.

![](images/27.png)

This is policy is also locked. Login as subscription Owner and try to disable this policy.

![](images/28.png)

Note this operation failed because policy is locked by Blueprint.

![](images/29.png)

Now as blueprint has been assigned we can remove Owner role from blueprintRobot as it is no longer needed and according to least privilege strategy we should remove this assignment.

```powershell
Set-AzContext -Subscription mojesub2
Remove-AzRoleAssignment -ObjectId 61895eee-e28b-428f-9b66-0cf0e6f5ce45 -RoleDefinitionName "Owner"
```

When adding other subscriptions follow the same procedure:
1. Make blueprintRobot Owner
2. Assign Blueprint
3. Remove blueprintRobot Owner role

Note identity blueprintRobot will be kept for all subsequent operations/subscriptions, but will be Owner of subscription just during blueprint assignment.