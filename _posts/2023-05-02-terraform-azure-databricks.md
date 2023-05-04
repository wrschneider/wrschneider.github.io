---
layout: post
title: Provisioning Azure Databricks workspace with Terraform
description: How to create an Azure Databricks workspace and associated resources with Terraform
---

Terraform provides an [`azurerm_databricks_workspace`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/databricks_workspace) resource to create an Azure Databricks
workspace.

The workspace itself is simple enough, but there are a number of other related resources
you may need to provision with Terraform to get a fully working environment.

### Private endpoints to storage

If you use VNet injection (also known as "no public IP") with Databricks, you will probably
want Databricks to use a private endpoint to access your storage account, to ensure a 
direct path from Databricks to your storage regardless of VNet egress configuration.

This can be provisioned through Terraform as well with the following resources:

* [`azurerm_private_endpoint`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/private_endpoint)
* `azurerm_private_dns_zone`
* `azurerm_private_dns_zone_virtual_network_link`

Note that there is nothing special about Databricks here, other than that you will need
a separate subnet for the private endpoint, in the same VNet.  You can't create the 
private endpoint in either of the two subnets you create with delegation to 
Databricks.

The other note is that, because storage accounts only allow _one_ subresource name
(despite the argument being a list in Terraform) you would need a separate endpoint if you
want both `blob` and `dfs` to work.

Once this is set up you can validate that Databricks is indeed hitting your private 
endpoint by running a shell command in a notebook like

```
%sh
ping mystorageaccount.dfs.core.windows.net
```

And see that the IP address is an IP in your private endpoint subnet.

### User-defined routes

Another best-practice is to use [User-Defined Routes (UDRs)](https://learn.microsoft.com/en-us/azure/databricks/administration-guide/cloud-configurations/azure/udr) with your Databricks VNet to provide the most direct path to the Databricks control plane.  Otherwise
there's a risk you may inadvertently ["trombone"](https://en.wikipedia.org/wiki/Anti-tromboning) through your on-prem network if you have VNet peering to an Express 
Route or VPN.

UDRs are not always strictly required, but if your VNet does not account for egress
to the control plane somehow, you may see timeout issues trying to launch new Databricks clusters.  At the very least, your VNet configuration needs to account for egress
to the control plane somehow.

### Secret management

Databricks Unity is the ideal way to connect to ADLS2 with managed identity.
If this isn't an option, you can connect Databricks to ADLS2 with a service principal
and secret.  The secrets can be stored in Azure Key Vault and Databricks can reference
the Key Vault secrets with a [secret scope](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes).

First, create Terraform resources to create the service principal and secret:

* `azuread_application`
* `azuread_service_principal` (references application above)
* `azuread_application_password` (references application above)
* `azurerm_key_vault_secret` (references secret and puts it in Key Vault)

Then grant the service principal access to the storage account:

* `azurerm_role_assignment` with "Storage Blob Data Reader/Contributor" (references storage account and service principal)

Sidebar: The distinction between "application" and "service principal" in Azure
can be confusing -- the concepts are linked but different.  The Application is the
resource with an OAuth client ID and secret and the service principal is what is granted
access to other Azure resources.

Once you have the Azure resources you can create a [secret scope in Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/secret-scopes#akv-ss).

Previously it was not possible to do this with the Databricks provider resource  
[`databricks_secret_scope`](https://registry.terraform.io/providers/databricks/databricks/latest/docs/resources/secret_scope) ("Currently, it's only possible to create Azure Key Vault scopes with Azure CLI authentication and not with Service Principal...")

[This may have changed recently.](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2023/april#create-an-azure-key-vault-backed-secret-scope-with-a-service-principal)

However, using the `databricks` Terraform provider needs to be done with caution,
as you can run into issues if you have direct dependencies between `azurerm_databricks_workspace` resources and the `databricks` provider.
[This is a general challenge with Terraform](https://stackoverflow.com/questions/75223275/terraform-providers-with-dependencies-on-resources) not limited to Databricks.


