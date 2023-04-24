---
layout: single
title:  "Kubernetes with kOps and Terraform"
date:   2023-04-24 20:00:00 -0400
---

Recently, we migrated our infrastructure from AWS ECS (Elastic Container Service) to our own Kubernetes cluster. Why we made that change is a post of its own, but a quick TL;DR would be:

- We needed scheduling capabilites - like K8s CronJobs - but ECS did not natively provide without adding dependency on another AWS Service.
- There wasn't a clear path about how to run DB migration scripts before each deploy
- We did not have a great experience with rolling updates on ECS. 
- I don't like to have dependencies to services from a specific cloud provider (AWS in this case), for multitude of reasons, the biggest one of which is the dependency it involves on proprietary software & tools.

_(A few of these might be on me as I'm not an ECS power user :P)_ 

<div class="align-center" style="width: 300px">
<div style="width:100%;height:0;padding-bottom:56%;position:relative;"><iframe src="https://giphy.com/embed/jnhXd7KT8UTk5WIgiV" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/minions-minions-2-rise-of-gru-jnhXd7KT8UTk5WIgiV">via GIPHY</a></p>
</div>

## kOps - Kubernetes Operations

Website / Docs: [https://kops.sigs.k8s.io/cli/kops/](https://kops.sigs.k8s.io/cli/kops/)

> kOps is Kubernetes Operations.
>
> kOps is the easiest way to get a production grade Kubernetes cluster up and running. We like to think of it as kubectl for clusters.

I've worked in the past with kOps and it really helped me maintain my infrastructure as code. I used kOps export files instead of relying on the CLI interface. What I did was:

- When creating clusters, use `--dry-run` and `--output` options to generate YAML config file for the cluster
```bash
kops create cluster \
    # Your kops create cluster options go here
    # --OPTION=VALUE
    --output yaml \
    --dry-run > my_cluster.yaml
```
- Generated YAML file was checked in version control (no it does not contain secrets)
- If I needed to change anything in the cluster, I would edit the YAML file, and do the following commands to update kOps' state:
```bash
kops replace -f my_cluster.yaml
kops update # verify the changes are what you expected
kops update --yes
kops rolling-update --yes # If needed
```

The above approach is a bit different from the default way to use kOps which involves using `kops edit cluster`, `kops edit ig nodes` and other commands to make the changes, without keeping track of them in version control via the YAML file.

The YAML file helps with making sure you can re-create the cluster if need be, without missing any important add-on or change. 

<p class="notice">
<strong>Example</strong> <br/>
In our case, one such thing was <code>httpPutResponseHopLimit</code> for our instances to be increased above the default value of 1. <br/>
<br/>

If we had made the change via <code>kops edit ig nodes</code>, I would for sure have forgotten that this was an important step, and if I ever had to re-create the cluster, I would definitely have a hard time recalling!
</p>

## Terraform

Website / Docs: [https://www.terraform.io/](https://www.terraform.io/)

> Terraform Cloud enables infrastructure automation for provisioning, compliance, and management of any cloud, datacenter, and service.

Terraform is a very popular infrastructure-as-a-code tool, and you can use it to automate setting up essentially any type of infrastructure on any major cloud provider. It is also able to reconcile cloud resources which have deviated from their desired state (deleted or updated manually by mistake). 

But hold on, doesn't kOps do the same for your infrastructure needed for your K8s cluster? kOps does do that, but I've found that terraform does a better job most of the time when it comes to managing cloud resources.

<p class="notice">
<strong>Example</strong> <br/>
When I previously used kOps, it had a hard time reconciling infrastructure differences, incase I had deleted / modified something on AWS directly.
</p>

Terraform also provides many additional utilities and scripting abilities which aren't there with kOps just yet. 

What's great though, is that kOps allows __exporting__ a Terraform config instead of creating / managing cloud resources itself, and this my fellow technologist, is the gold mine!

<div class="align-center" style="width: 300px">
<div style="width:100%;height:0;padding-bottom:75%;position:relative;"><iframe src="https://giphy.com/embed/pPzjpxJXa0pna" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/pPzjpxJXa0pna">via GIPHY</a></p>
</div>

## kOps X Terraform

![Kowalski, Analysis!](https://i.kym-cdn.com/entries/icons/original/000/037/334/Kowalski.jpg)

To use Terraform for managing your cloud resources, add the `--out` and `--target` options to your `kops update cluster` command:

```bash
kops update cluster --out=kops-terraform --target=terraform
```

This will generate terraform config files under the `kops-terraform` directory. You can change into that directory and run `terraform apply`, and terraform will create the cloud resources for you.

For our use case, however, we needed some extra resources to be created, such as a VPC Peering Connection, some extra IAM Roles and Policies. These resources were used during runtime - by pods running in k8s cluster to use AWS APIs.

In order to achieve that, I created a new terraform module, and imported the terraform generated by kOps as a module. Here's how you can do that:

```terraform
# terraform.tf

module "kops" {
  source = "./kops-terraform"
}


resource "aws_iam_role" "my_custom_role" {
  inline_policy {
    name = "my_role_policy"
    policy = jsonencode({
      "Version": "2012-10-17",
      "Statement": [
        "Your IAM Role Policy Statements"
      ]
    })
  }
}
```

When I run `terraform apply` in my directory, it creates the resources needed by kOps, AND, my other resources too.

### Handling cluster updates with Terraform

Handling kOps cluster updates is a bit different when using kOps with Terraform vs when using kOps only. When using terraform, you need to run these commands after updating your cluster YAML file:


```bash
kops replace -f my_cluster.yaml

# Check that the change is what you expected
kops update 

# Generate terraform config, this will overwrite the kops-terraform directory
kops update --yes --out=kops-terraform --target=terraform 

# Update cloud resources
terraform apply
```

<section class="notice notice-warning">
<p><b>Note: Don't modify terraform files!</b></p>
<p>kOps will update terraform files written in the output directory, you must consider kOps as the single source of truth and not modify the generated terraform files manually.</p>

<p>This is why I suggest including the kOps terraform files as module and adding any additional cloud resource configuration in your own terraform files.</p>
</section>

## Bonus: Using Terraform output in Helmfile

Terraform offers an option to output the terraform output as a JSON which you can load in your Helmfile setup. This is very useful if you want to provide resource identifiers which are usually dynamically generated when the cloud resources are created.

To generate the terraform output (advisable to gitignore this):
```bash
terraform output -json > terraform.json
```

And then in your `helmfile.yaml`, you can do:
```go
values:
    - terraform: {{ "{{ " }} readFile "./terraform.json" }}
```
<div class="align-center" style="width: 300px">
<div style="width:100%;height:0;padding-bottom:125%;position:relative;"><iframe src="https://giphy.com/embed/l2Sq5GffrCyUMEXjW" width="100%" height="100%" style="position:absolute" frameBorder="0" class="giphy-embed" allowFullScreen></iframe></div><p><a href="https://giphy.com/gifs/arg-l2Sq5GffrCyUMEXjW">via GIPHY</a></p>
</div>