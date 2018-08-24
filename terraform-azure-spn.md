# Beyond Infrastructure Part 3 - Using Terraform to Service Principal Creation

In parts 1 and 2 of the Beyond Infrastructure series we looked at how Terraform can manage Azure Policies and Azure Locks; in part 3 we will take a look at generating Service Principals.

Intro statements.

## Background

what is an SPN?
links

## Traditional Approach

Portal, CLI, other??

Can Terraform do this more easily?

## Azure Terraform Provider

### azurerm_azuread_application

### azurerm_azuread_service_principal

### azurerm_role_assignment


## How to

To get started we will need to create a AD Application.

```hcl
resource "azurerm_azuread_application" "main" {
  name = "myapplicationname"
}
```

Next we need the Service Principal associated with the AD Application.

```hcl
resource "azurerm_azuread_service_principal" "main" {
  application_id = "${azurerm_azuread_application.main.application_id}"
}
```

At this point if we were to apply these resources two things would be true:

- We would not know the Service Principle password
- The Service Principle would not have access to anything in your subscription

Let us address both of these by adding two more resources.

```hcl
resource "azurerm_azuread_service_principal_password" "main" {
  service_principal_id = "${azurerm_azuread_service_principal.main.id}"
  value                = "SuperSecretPassword"
  end_date             = "${local.spn_expire}"
}

resource "azurerm_role_assignment" "main" {
  scope                = "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  role_definition_name = "Contributor"
  principal_id         = "${azurerm_azuread_service_principal.main.id}"
}
```

Adding a `azurerm_azuread_service_principal_password` we are able to assign a specific password to the Service Principal and `azurerm_role_assignment` will assign that Service Principal contributor access to the entire subscription.

If we take these resources, apply a little refactoring, we end up with the following:

```hcl
data "azurerm_subscription" "primary" {}

resource "random_string" "password" {
  length  = 32
  special = true
}

locals {
  spn_name         = "tfexample"
  spn_expire       = "2020-01-01T00:00:00Z"
  spn_password     = "${random_string.password.result}"
  spn_subscription = "${data.azurerm_subscription.primary.id}"
}

resource "azurerm_azuread_application" "main" {
  name = "${local.spn_name}"
}

resource "azurerm_azuread_service_principal" "main" {
  application_id = "${azurerm_azuread_application.main.application_id}"
}

resource "azurerm_azuread_service_principal_password" "main" {
  service_principal_id = "${azurerm_azuread_service_principal.main.id}"
  value                = "${local.spn_password}"
  end_date             = "${local.spn_expire}"
}

resource "azurerm_role_assignment" "main" {
  scope                = "${local.spn_subscription}"
  role_definition_name = "Contributor"
  principal_id         = "${azurerm_azuread_service_principal.main.id}"
}

output "spn.id" {
  value = "${azurerm_azuread_service_principal.main.application_id}"
}

output "spn.password" {
  value = "${random_string.password.result}"
}
```

> Note: We use the `azurerm_subscription` data source to get the current subscription id Terraform is using to execute and `random_string` to generate a random password. Both of these are for ease of demonstration and how you manage secrets is a deeper topic, take a look at [Vault](https://www.hashicorp.com/products/vault) for an Enterprise ready solution.

## Real World Example

A great use case for this type of Service Principal Management would be using the [Managed Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). When you create an AKS cluster, one of the requirements is a Service Principal that the AKS cluster uses to dynamically manage underlying infrastructure.

Building on the solution above, we can manage the Service Principal with the same Terraform as the AKS cluster, reducing the overhead and potential clean up needed when the cluster is deleted.

```hcl
resource "tls_private_key" "main" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "azurerm_resource_group" "test" {
  name     = "acctestRG1"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "test" {
  name                = "acctestaks1"
  location            = "${azurerm_resource_group.test.location}"
  resource_group_name = "${azurerm_resource_group.test.name}"
  dns_prefix          = "acctestagent1"

  linux_profile {
    admin_username = "acctestuser1"

    ssh_key {
      key_data = "${tls_private_key.main.public_key_openssh}"
    }
  }

  agent_pool_profile {
    name            = "default"
    count           = 1
    vm_size         = "Standard_D1_v2"
    os_type         = "Linux"
    os_disk_size_gb = 30
  }

  service_principal {
    client_id     = "${azurerm_azuread_service_principal.main.application_id}"
    client_secret = "${azurerm_azuread_service_principal_password.main.value}"
  }

  tags {
    Environment = "Production"
  }
}
```


## Conclusions

Easily create and manage the SPN.
Use the SPN for other TF resources.
Could be used to just manage SPNs and applied to a work flow.



<!-- In this post we have shown how you can leverage Terraform to manage Azure Locks to protect critical resources from deletion/modification in your Azure Subscription using exist or new Terraform configuration. Leveraging the Terraform workflow results in a source code driven solution that can be change controlled, versioned, and visible to your entire organization. -->

<!-- > Stay tuned for the next blog in the "Beyond Infrastructure" series! -->

All assets in this blog post can be found in the following [Gist](https://gist.github.com/straubt1/xxx)

## Next Steps

To learn more about the benefits of Infrastructure as Code using Terraform, contact us at [info@cardinalsolutions.com](mailto:info@cardinalsolutions.com). From our 1-day hands-on workshop, to a 1-week guided pilot, we can help your organization implement or migrate to an Infrastructure as Code architecture for managing your cloud infrastructure.