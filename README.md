# terraform-aws-cf-install

This project aims to create one click deploy for Cloud Foundry into an AWS VPC.


## Architecture
We rely on two other repositories to do the bulk of the work. The [terraform-aws-vpc](https://github.com/cloudfoundry-community/terraform-aws-vpc)
repo creates the base VPC infrastructure, including a bastion subnet, the `microbosh` subnet, a NAT server,
various route tables, and the VPC itself. Then, the [terraform-aws-cf-net](https://github.com/cloudfoundry-community/terraform-aws-cf-net) repo
creates a loadbalancer subnet, two runtime subnets, `cf` related security groups, and
the elastic IP used by `cf`. This gives us the flexibility to use the `terraform-aws-cf-net`
module multiple times, to have a staging and production cf within the same VPC,
sharing a single microbosh instance.

## Deploy Cloud Foundry

### Prerequisites

The one step that isn't automated is the creation of SSH keys. We are waiting for
that feature to be [added to terraform](https://github.com/hashicorp/terraform/issues/28).
An AWS SSH Key need to be created in desired region prior to running the following
commands. Note the name of the key and the path to the pem/private key file for
use further down.

You **must** being using at least terraform version 0.3.6.

Your chosen AWS Region must have sufficient quota to spin up all of the machines.
While building various bits, the install process can use up to 13 VMs, settling
down to use 7 machines long-term (more, if you want more runners).

Optionally for using the `Unattended Install` instruction, install git.

### Easy install
```bash
mkdir terraform-cf-install
cd terraform-cf-install
terraform apply github.com/cloudfoundry-community/terraform-aws-cf-install
```

### Unattended install
```bash
git clone https://github.com/cloudfoundry-community/terraform-aws-cf-install
cp terraform.tfvars-example terraform.tfvars
```

Next, edit terraform.tfvars using your favorite editor (`vim`), filling out the
variables with your own values.

```bash
make plan
make apply
```

### Cleanup / Tear down
Terraform does not yet quite cleanup after itself. You can run `make destroy` to
get quite a few of the resources you created, but you will probably have to manually
track down some of the bits and manually remove them. Once you have done so, run `make clean`
to remove the local cache and status files, allowing you to run everything again
without errors.

## Module Outputs
If you wish to use this module in conjunction with `terraform-aws-cf-net` to create
more than one `cf` instance in a single VPC, that is fully supported. The following
variables are outputs of the module, suitable to be used as variable inputs to the
`terraform-aws-cf-net` module:

```
aws_vpc_id
aws_internet_gateway_id
aws_route_table_public_id
aws_route_table_private_id
aws_subnet_bastion_availability_zone
```

### Example usage

Note that this does not actually create the second `cf` instance, that has to be
done manually. You should be able to take the resources created by the `cf-staging` 
module, copy the cf-boshbootstrap directory on the bastion server, and search and
replace with the new values. Also, you can set the `offset` value to whatever you
want, from 1 to 24.

```
provider "aws" {
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
  region = "${var.aws_region}"
}

module "cf-install" {
  source = "github.com/cloudfoundry-community/terraform-aws-cf-install"
  network = "${var.network}"
  aws_key_name = "${var.aws_key_name}"
  aws_access_key = "${var.aws_access_key}"
  aws_secret_key = "${var.aws_secret_key}"
  aws_region = "${var.aws_region}"
  aws_key_path = "${var.aws_key_path}"
}

module "cf-staging" {
  source = "github.com/cloudfoundry-community/terraform-aws-cf-net"
  network = "${var.network}"
  aws_key_name = "${var.aws_key_name}"
  aws_access_key = "${var.aws_access_key}"
  aws_secret_key = "${var.aws_secret_key}"
  aws_region = "${var.aws_region}"
  aws_key_path = "${var.aws_key_path}"
  aws_vpc_id = "${module.cf-install.aws_vpc_id}"
  aws_internet_gateway_id = "${module.cf-install.aws_internet_gateway_id}"
  aws_route_table_public_id = "${module.cf-install.aws_route_table_public_id}"
  aws_route_table_private_id = "${module.cf-install.aws_route_table_private_id}"
  aws_subnet_lb_availability_zone = "${module.cf-install.aws_subnet_bastion_availability_zone}"
  offset = "20"
}
```