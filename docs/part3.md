<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 3: Application

## Steps

### 1. Terraform init

The `terraform init` command initializes a working directory containing Terraform configuration files.

```sh
% terraform init
```

Expected output:

<!-- SCREENSHOT -->

### 1. Terraform plan

The `terraform plan` command creates an execution plan, which lets you preview the changes that Terraform plans to make to your infrastructure.

```sh
% terraform plan
```

Expected output:

<!-- SCREENSHOT -->

### 2. Terraform apply

The `terraform apply` command executes the actions proposed in a Terraform plan.

```sh
% terraform apply
```

Enter yes when prompted for input

Expected output:

<!-- SCREENSHOT -->

### 3. Verify

Take the `project_id` from the terraform apply and use metal cli to check the status of the new server

```
metal devices get --project-id $(terraform output project_id)
```

Expected output:

<!-- SCREENSHOT -->


SSH login into the server

```
ssh -i ~/.ssh/equinix-metal-terraform-rsa root@$(terraform output device_public_ip)
```

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* How Terraform keeps track of my infrastructure changes?
* Can I scale up or replicate my device to another location without duplicating code?
