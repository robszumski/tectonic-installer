# Install Tectonic on Azure with Terraform

Following this guide will deploy a Tectonic cluster within your Azure account.

Generally, the Azure platform templates adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to document the implementation details specific to the Azure platform.

## Prerequsities

 - **DNS** - Setup your DNS zone in a resource group called `tectonic-dns-group` or specify a different resource group using the `tectonic_azure_dns_resource_group` variable below. We use a separate resource group assuming that you have a zone that you already want to use. Follow the [docs to set one up][azure-dns].
 - **Make** - This guide uses `make` to download a customized version of Terraform, which is pinned to a specific version and includes required plugins.
 - **Tectonic Account** - Register for a [Tectonic Account][register], which is free for up to 10 nodes. You will need to provide the cluster license and pull secret below.

## Getting Started

First, clone the Tectonic Installer repository in a convenient location:

```
$ git clone https://github.com/coreos/tectonic-installer.git
$ git checkout tags/tectonic-1.5.5-tectonic.3
```

Download the pinned Terraform binary and modules required for Tectonic:

```
$ wget https://releases.tectonic.com/tectonic-1.5.5-tectonic.3.tar.gz
$ tar xzvf tectonic-1.5.5-tectonic.3.tar.gz
$ cd tectonic/tectonic-installer
```

After downloading, you will need to source this new binary in your `$PATH`. This is important, especially if you have another verison of Terraform installed. Run this command to add it to your path:

```
$ export PATH=/path/to/tectonic-installer/bin/terraform:$PATH
```

You can double check that you're using the binary that was just downloaded:

```
$ which terraform
/Users/coreos/tectonic-installer/bin/terraform/terraform
```

Next, get the modules that Terraform will use to create the cluster resources:

```
$ terraform get platforms/azure
Get: file:///Users/tectonic-installer/modules/azure/vnet
Get: file:///Users/tectonic-installer/modules/azure/etcd
Get: file:///Users/tectonic-installer/modules/azure/master
Get: file:///Users/tectonic-installer/modules/azure/worker
Get: file:///Users/tectonic-installer/modules/azure/dns
Get: file:///Users/tectonic-installer/modules/bootkube
Get: file:///Users/tectonic-installer/modules/tectonic
```

Generate credentials using the Azure CLI. If you're not logged in, execute `az login` first. See the [docs][login] for more info.

```
$ az ad sp create-for-rbac -n "http://tectonic" --role contributor
Retrying role assignment creation: 1/24
Retrying role assignment creation: 2/24
{
 "appId": "generated-app-id",
 "displayName": "azure-cli-2017-01-01",
 "name": "http://tectonic-coreos",
 "password": "generated-pass",
 "tenant": "generated-tenant"
}
```

Export variables that correspond to the data that was just generated. The subscription is your Azure Subscription ID.

```
$ export ARM_SUBSCRIPTION_ID=abc-123-456
$ export ARM_CLIENT_ID=generated-app-id
$ export ARM_CLIENT_SECRET=generated-pass
$ export ARM_TENANT_ID=generated-tenant
```

Now we're ready to specify our cluster configuration.

## Customize the deployment

Customizations to the base installation live in `platforms/azure/terraform.tfvars.example`. Export a variable that will be your cluster identifier:

```
$ export CLUSTER=my-cluster
```

Create a build directory to hold your customizations and copy the example file into it:

```
$ mkdir -p build/${CLUSTER}
$ cp platforms/azure/terraform.tfvars.example build/${CLUSTER}/terraform.tfvars
```

 - **tectonic_base_domain** - domain name that is set up with in a resource group, as described in the prerequisites.
 - **tectonic_pull_secret_path** - path on disk to your downloaded pull secret. You can find this on your [Account dashboard][account].
 - **tectonic_license_path** - path on disk to your downloaded Tectonic license. You can find this on your [Account dashboard][account].
 - **tectonic_admin_password_hash** - generate a hash with the [bcrypt-hash tool][bcrypt] that will be used for your admin user.

## Deploy the cluster

Test out the plan before deploying everything:

```
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

Next, deploy the cluster:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

This should run for a little bit, and when complete, your Tectonic cluster should be ready.

If you encounter any issues, check the known issues and workarounds below.

To delete your cluster, run:

```
$ terraform destroy -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

### Known issues and workarounds

See the [troubleshooting][troubleshooting] document for work arounds for bugs that are being tracked.

## Scaling the cluster

To scale worker nodes, adjust `tectonic_worker_count` in `terraform.vars` and run:

```
$ terraform apply $ terraform plan \
  -var-file=build/${CLUSTER}/terraform.tfvars \
  -target module.workers \
  platforms/azure
```

## Under the hood

### Top-level templates

* The top-level templates that invoke the underlying component modules reside `./platforms/azure`
* Point terraform to this location to start applying: `terraform apply ./platforms/azure`

### Etcd nodes

* Discovery is currently not implemented so DO NOT scale the etcd cluster to more than 1 node, for now.
* Etcd cluster nodes are managed by the terraform module `modules/azure/etcd`
* Node VMs are created as stand-alone instances (as opposed to VM scale sets). This is mostly historical and could change.
* A load-balancer fronts the etcd nodes to provide a simple discovery mechanism, via a VIP + DNS record.
* Currently, the LB is configured with a public IP address. This is not optimal and it should be converted to an internal LB.

### Master nodes

* Master node VMs are managed by the templates in `modules/azure/master`
* An Azure VM Scaling Set resource is used to spin-up multiple identical VM configured as master nodes.
* Master VMs all share the same identical Ignition config
* Master nodes are fronted by one load-balancer for the API one for the Ingress controller.
* The API LB is configured with SourceIP session stickiness, to ensure that TCP (including SSH) sessions from the same client land reliably on the same master node. This allows for provisioning the assets and starting bootkube reliably via SSH.
* a `null_resource` terraform provisioner in the tectonic.tf top-level template will copy the assets and run bootkube automatically on one of the masters.
* make sure the SSH key specifyied in the tfvars file is also added to the SSH agent on the machine running terraform. Without this, terraform is not able to SSH copy the assets and start bootkube. Also make sure that the SSH known_hosts file doesn't have old records of the API DNS name (fingerprints will not match).

### Worker nodes

* Worker node VMs are managed by the templates in `modules/azure/worker`
* An Azure VM Scaling Set resource is used to spin-up multiple identical VM configured as worker nodes.
* Worker VMs all share the same identical Ignition config
* Worker nodes are not fronted by any LB and don't have public IP addresses. They can be accessed through SSH from any of the master nodes.

[conventions]: ../conventions.md
[generic]: ../generic-platform.md
[register]: https://account.tectonic.com/signup/summary/tectonic-2016-12
[account]: https://account.tectonic.com
[bcrypt]: https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0
[plan-docs]: https://www.terraform.io/docs/commands/plan.html
[copy-docs]: https://www.terraform.io/docs/commands/apply.html
[troubleshooting]: ../troubleshooting.md
[login]: https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli
[azure-dns]: https://docs.microsoft.com/en-us/azure/dns/dns-getstarted-portal