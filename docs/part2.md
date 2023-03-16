<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 2: Provision

## Terraform Project’s Structure

The project structure in terraform is quite flexible. However, we do believe that a good structure and naming rules is essential to guarantee its correct maintenance in daily operations

Code in the Terraform language is stored in plain text files with the .tf file extension. You can keep all your code in a single main.tf file with hundreds of lines of code, or split them in several well named files and folders that make sense for your use case (this is the correct option).  Below is an example of a simple project structure:

* main.tf Declare all resource and datasource definitions
* outputs.tf Declare all outputs in this file. Output values are similar to return values in programming languages
* variables.tf Declare all input variables in this file. Input variables let you customize aspects of Terraform modules without altering the module's own source code
* terraform.tfvars Assign or override values of the variables defined in the variables.tf file

## Steps

### 1. Provider

Create a `main.tf` file to configure the provider. Insert the code below and keep the `auth_token` as is, we will update it later in this workshop

```hcl
terraform {
  required_providers {
    equinix = {
      source = "equinix/equinix"
    }
  }
}

# Credentials for Equinix Metal resources
provider "equinix" {
  auth_token = "someEquinixMetalToken"

  ## client_id and client_secret can be omitted when the only
  ## Equinix service consumed are Equinix Metal resources
  # client_id     = "someEquinixAPIClientID"
  # client_secret = "someEquinixAPIClientSecret"
}
```

### 2. Resources

Define a new metal project

```hcl
resource "equinix_metal_project" "project" {
  name = "Terraform Workshop"
}
```

Add a new Equinix Metal device (baremetal server) with an implicit dependency in `project_id`

```hcl
resource "equinix_metal_device" "device" {
  hostname         = "test-device"
  plan             = "c3.small.x86"
  metro            = "sv"
  operating_system = "ubuntu_20_04"
  billing_cycle    = "hourly"
  project_id       = equinix_metal_project.project.id
}
```

Add a new Equinix Metal project SSH key 

```hcl
resource "equinix_metal_project_ssh_key" "public_key" {
  name       = "terraform-rsa"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAAD..."
  project_id = equinix_metal_project.project.id
}
```

Use terraform `TLS` provider to create a PEM (and OpenSSH) formatted private key.

```hcl
resource "tls_private_key" "ssh_key_pair" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
```

> **_Pro Tip:_** unless you need a specific provider version, none of the official [hashicorp providers](https://registry.terraform.io/namespaces/hashicorp) need to be added to the `terraform.required_providers` block

Use terraform `local` provider to store the private key

```hcl
resource "local_file" "private_key" {
  content         = chomp(tls_private_key.ssh_key_pair.private_key_pem)
  filename        = pathexpand(format("~/.ssh/%s", "equinix-metal-terraform-rsa"))
  file_permission = "0600"
}
```

Update `equinix_metal_project_ssh_key.public_key` to reference the public key

```hcl
resource "equinix_metal_project_ssh_key" "public_key" {
  ...
  public_key = chomp(tls_private_key.ssh_key_pair.public_key_openssh)
}
```

> **_Pro Tip:_** you can use "depends_on" meta-argument to declare explicit dependencies between resources
 
If you create a new device in a project, all the keys of the project's collaborators will be injected to the device. Add `depends_on` in the device resource to make sure the key is created before the device

```hcl
resource "equinix_metal_device" "device" {
  ...
  depends_on = ["equinix_metal_project_ssh_key.public_key"]
}
```

### 4. Outputs

Create an `outputs.tf` file to expose some computed attributes that you will need later

```hcl
output "project_id" {
  value = equinix_metal_project.project.id
}

output "device_id" {
  value = equinix_metal_device.device.id
}

output "device_public_ip" {
  value = equinix_metal_device.device.access_public_ipv4
} 
```

### 5. Variables

Create a `variables.tf` file and define some useful inputs

```hcl
variable "plan" {
  type        = string
  description = "device type/size"
  default     = "c3.small.x86"
}

variable "location" {
  type        = string
  description = "device type/size"
  default     = "SV"
}

variable "os" {
  type        = string
  description = "device type/size"
  default     = "ubuntu_20_04"
}
```

Update `main.tf` to start using these new variables

```
resource "equinix_metal_device" "device" {
  hostname         = "test-device"
  plan             = var.plan
  metro            = var.location
  operating_system = var.os
  billing_cycle    = "hourly"
  project_id       = equinix_metal_project.project.id
  depends_on       = ["equinix_metal_project_ssh_key.public_key"]
}
```

### 6. Tfvars files

Create a `terraform.tfvars` file

Add key/value inputs for the cluster module

```
plan     = "c3.medium.x86"
location = "the-org-name-you-created"
metro    = "someone@companyx.com"
```

### 6. Environment variables

It is not a good practice to include your credentials directly in your template. Although there are more secure options, a good first practice is to use environment variables instead. Before using a new provider, checkout its documentation for more details on the [available authentication methods](https://registry.terraform.io/providers/equinix/equinix/latest/docs).

```sh
export METAL_AUTH_TOKEN=someEquinixMetalToken
# export EQUINIX_API_CLIENTID=someEquinixAPIClientID
# export EQUINIX_API_CLIENTSECRET=someEquinixAPIClientSecret
```

Remove `auth_token = "someEquinixMetalAPIToken"` line from `versions.tf` file

### 7. Verify

At this point, your project should look like this:

<!-- [[ADD IMAGE OF THE PROJECT TREE ]] -->

And the main.tf file:

<!-- [[ADD IMAGE OF THE MAIN FILE ]] -->

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Can other providers be used in the same template/project?
* Are all the Equinix platform resources available on terraform?
