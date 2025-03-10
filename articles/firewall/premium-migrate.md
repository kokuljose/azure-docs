---
title: Migrate to Azure Firewall Premium
description: Learn how to migrate from Azure Firewall Standard to Azure Firewall Premium.
author: vhorne
ms.service: firewall
services: firewall
ms.topic: how-to
ms.date: 12/02/2021
ms.author: victorh 
ms.custom: devx-track-azurepowershell
---

# Migrate to Azure Firewall Premium

You can migrate Azure Firewall Standard to Azure Firewall Premium to take advantage of the new Premium capabilities. For more information about Azure Firewall Premium features, see [Azure Firewall Premium features](premium-features.md).

The following two examples show how to:
- Migrate an existing standard policy using Azure PowerShell
- Migrate an existing standard firewall (with classic rules) to Azure Firewall Premium  with a Premium policy.

If you use Terraform to deploy the Azure Firewall, you can use Terraform to migrate to Azure Firewall Premium. For more information, see [Migrate Azure Firewall Standard to Premium using Terraform](/azure/developer/terraform/firewall-upgrade-premium?toc=/azure/firewall/toc.json&bc=/azure/firewall/breadcrumb/toc.json).

## Performance considerations

Performance is a consideration when migrating from the standard SKU. IDPS and TLS inspection are compute intensive operations. The premium SKU uses a more powerful VM SKU which scales to a maximum throughput of 30 Gbps comparable with the standard SKU. The 30 Gbps throughput is supported when configured with IDPS in alert mode. Use of IDPS in deny mode and TLS inspection increases CPU consumption. Degradation in max throughput might occur. 

The firewall throughput might be lower than 30 Gbps when you have one or more signatures set to **Alert and Deny** or application rules with **TLS inspection** enabled. Microsoft recommends customers perform full scale testing in their Azure deployment to ensure the firewall service performance meets your expectations.

## Downtime

Migrate your firewall during a planned maintenance time, as there will be some downtime during the migration.

## Migrate Classic rules to Standard policy

During your migration process, you may need to migrate your Classic firewall rules to a Standard policy. You can do this using the Azure portal:

1. From the Azure portal, select your standard firewall. On the **Overview** page, select **Migrate to firewall policy**.

   :::image type="content" source="media/premium-migrate/firewall-overview-migrate.png" alt-text="Migrate to firewall policy":::

1. On the **Migrate to firewall policy** page, select **Review + create**.
1. Select **Create**.

   The deployment takes a few minutes to complete.

You can also migrate existing Classic rules from Azure Firewall using Azure PowerShell to create policies. For more information, see [Migrate Azure Firewall configurations to Azure Firewall policy using PowerShell](../firewall-manager/migrate-to-policy.md)

## Migrate an existing policy using Azure PowerShell

`Transform-Policy.ps1` is an Azure PowerShell script that creates a new Premium policy from an existing Standard policy.

Given a standard firewall policy ID, the script transforms it to a Premium Azure Firewall policy. The script first connects to your Azure account, pulls the policy, transforms/adds various parameters, and then uploads a new Premium policy. The new premium policy is named `<previous_policy_name>_premium`. In case of child policy transformation, link to parent policy will remain.

Usage example:

`Transform-Policy -PolicyId /subscriptions/XXXXX-XXXXXX-XXXXX/resourceGroups/some-resource-group/providers/Microsoft.Network/firewallPolicies/policy-name`

> [!IMPORTANT]
> The script doesn't migrate Threat Intelligence settings. You'll need to note those settings before proceeding and migrate them manually.

```azurepowershell
<#
    .SYNOPSIS
        Given an Azure firewall policy id the script will transform it to a Premium Azure firewall policy. 
        The script will first pull the policy, transform/add various parameters and then upload a new premium policy. 
        The created policy will be named <previous_policy_name>_premium if no new name provided else new policy will be named as the parameter passed.  
    .Example
        Transform-Policy -PolicyId /subscriptions/XXXXX-XXXXXX-XXXXX/resourceGroups/some-resource-group/providers/Microsoft.Network/firewallPolicies/policy-name -NewPolicyName <optional param for the new policy name>
#>

param (
    #Resource id of the azure firewall policy. 
    [Parameter(Mandatory=$true)]
    [string]
    $PolicyId,

    #new filewallpolicy name, if not specified will be the previous name with the '_premium' suffix
    [Parameter(Mandatory=$false)]
    [string]
    $NewPolicyName = ""
)
$ErrorActionPreference = "Stop"
$script:PolicyId = $PolicyId
$script:PolicyName = $NewPolicyName

function ValidatePolicy {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [Object]
        $Policy
    )

    Write-Host "Validating resource is as expected"

    if ($null -eq $Policy) {
        Write-Error "Recived null policy"
        exit(1)
    }
    if ($Policy.GetType().Name -ne "PSAzureFirewallPolicy") {
        Write-Host "Resource must be of type Microsoft.Network/firewallPolicies" -ForegroundColor Red
        exit(1)
    }

    if ($Policy.Sku.Tier -eq "Premium") {
        Write-Host "Policy is already premium"
        exit(1)
    }
}

function GetPolicyNewName {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [Microsoft.Azure.Commands.Network.Models.PSAzureFirewallPolicy]
        $Policy
    )

    if (-not [string]::IsNullOrEmpty($script:PolicyName)) {
        return $script:PolicyName
    }

    return $Policy.Name + "_premium"
}

function TransformPolicyToPremium {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [Microsoft.Azure.Commands.Network.Models.PSAzureFirewallPolicy]
        $Policy
    )    
    $NewPolicyParameters = @{
                        Name = (GetPolicyNewName -Policy $Policy) 
                        ResourceGroupName = $Policy.ResourceGroupName 
                        Location = $Policy.Location 
                        ThreatIntelMode = $Policy.ThreatIntelMode 
                        BasePolicy = $Policy.BasePolicy.Id 
                        DnsSetting = $Policy.DnsSettings 
                        Tag = $Policy.Tag 
                        SkuTier = "Premium" 
    }

    Write-Host "Creating new policy"
    $premiumPolicy = New-AzFirewallPolicy @NewPolicyParameters

    Write-Host "Populating rules in new policy"
    foreach ($ruleCollectionGroup in $Policy.RuleCollectionGroups) {
        $ruleResource = Get-AzResource -ResourceId $ruleCollectionGroup.Id
        $ruleToTransfom = Get-AzFirewallPolicyRuleCollectionGroup -AzureFirewallPolicy $Policy -Name $ruleResource.Name
        $ruleCollectionGroup = @{
            FirewallPolicyObject = $premiumPolicy
            Priority = $ruleToTransfom.Properties.Priority
            Name = $ruleToTransfom.Name
        }

        if ($ruleToTransfom.Properties.RuleCollection.Count) {
            $ruleCollectionGroup["RuleCollection"] = $ruleToTransfom.Properties.RuleCollection
        }

        Set-AzFirewallPolicyRuleCollectionGroup @ruleCollectionGroup
    }
}

function ValidateAzNetworkModuleExists {
    Write-Host "Validating needed module exists"
    $networkModule = Get-InstalledModule -Name "Az.Network" -MinimumVersion 4.5 -ErrorAction SilentlyContinue
    if ($null -eq $networkModule) {
        Write-Host "Please install Az.Network module version 4.5.0 or higher, see instructions: https://github.com/Azure/azure-powershell#installation"
        exit(1)
    }
    $resourceModule = Get-InstalledModule -Name "Az.Resources" -MinimumVersion 4.2 -ErrorAction SilentlyContinue
    if ($null -eq $resourceModule) {
        Write-Host "Please install Az.Resources module version 4.2.0 or higher, see instructions: https://github.com/Azure/azure-powershell#installation"
        exit(1)
    }
    Import-Module Az.Network -MinimumVersion 4.5.0
    Import-Module Az.Resources -MinimumVersion 4.2.0
}

ValidateAzNetworkModuleExists
$policy = Get-AzFirewallPolicy -ResourceId $script:PolicyId
ValidatePolicy -Policy $policy
TransformPolicyToPremium -Policy $policy

```

## Migrate Azure Firewall using stop/start

If you use Azure Firewall Standard SKU with Firewall Policy, you can use the Allocate/Deallocate method to migrate your Firewall SKU to Premium. This migration approach is supported on both VNet Hub and Secure Hub Firewalls. When you migrate a Secure Hub deployment, it will preserve the firewall public IP address.

The minimum Azure PowerShell version requirement is 6.5.0. For more information, see [Az 6.5.0](https://www.powershellgallery.com/packages/Az/6.5.0).

 
### Migrate a VNET Hub Firewall

- Deallocate the Standard Firewall 

   ```azurepowershell
   $azfw = Get-AzFirewall -Name "<firewall-name>" -ResourceGroupName "<resource-group-name>"
   $azfw.Deallocate()
   Set-AzFirewall -AzureFirewall $azfw
   ```


- Allocate Firewall Premium

   ```azurepowershell
   $azfw = Get-AzFirewall -Name "<firewall-name>" -ResourceGroupName "<resource-group-name>"
   $azfw.Sku.Tier="Premium"
   $vnet = Get-AzVirtualNetwork -ResourceGroupName "<resource-group-name>" -Name "<Virtual-Network-Name>"
   $publicip = Get-AzPublicIpAddress -Name "<Firewall-PublicIP-name>" -ResourceGroupName "<resource-group-name>"
   $azfw.Allocate($vnet,$pip)
   Set-AzFirewall -AzureFirewall $azfw
   ```

- Allocate Firewall Premium in Forced Tunnel Mode

   ```azurepowershell
   $azfw = Get-AzFirewall -Name "<firewall-name>" -ResourceGroupName "<resource-group-name>"
   $azfw.Sku.Tier="Premium"
   $vnet = Get-AzVirtualNetwork -ResourceGroupName "<resource-group-name>" -Name "<Virtual-Network-Name>"
   $publicip = Get-AzPublicIpAddress -Name "<Firewall-PublicIP-name>" -ResourceGroupName "<resource-group-name>"
   $mgmtPip = Get-AzPublicIpAddress -ResourceGroupName "<resource-group-name>"-Name "<Management-PublicIP-name>"
   $azfw.Allocate($vnet,$publicip,$mgmtPip)
   Set-AzFirewall -AzureFirewall $azfw
   ```

### Migrate a Secure Hub Firewall


- Deallocate the Standard Firewall

   ```azurepowershell
   $azfw = Get-AzFirewall -Name "<firewall-name>" -ResourceGroupName "<resource-group-name>"
   $azfw.Deallocate()
   Set-AzFirewall -AzureFirewall $azfw
   ```

- Allocate Firewall Premium

   ```azurepowershell
   $azfw = Get-AzFirewall -Name -Name "<firewall-name>" -ResourceGroupName "<resource-group-name>"
   $hub = get-azvirtualhub -ResourceGroupName "<resource-group-name>" -name "<vWAN-name>"
   $azfw.Sku.Tier="Premium"
   $azfw.Allocate($hub.id)
   Set-AzFirewall -AzureFirewall $azfw
   ```

## Attach a Premium policy to a Premium Firewall

You can attach a Premium policy to the new Premium Firewall using the Azure portal:

:::image type="content" source="media/premium-migrate/premium-firewall-policy.png" alt-text="Screenshot showing firewall policy":::

## Next steps

- [Learn more about Azure Firewall Premium features](premium-features.md)
