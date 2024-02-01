# mtc-dev-repo-README
This is a README for the terraform cloud deployment of the compute and networking repos terraform-aws-networking and terraform-aws-compute. The terraform cloud configuration designer is used to create the main.tf which is deployed to a real time mtc-dev-repo created on terraform apply of a main.tf controller file on Cloud9 IDE.  

This is the controller main.tf file on Cloud9 IDE that orchestrates this.  Note that the deployment github repo is exclusively created here on terraform (not manually through the github web console). Likewise the oAuth is set up through this terraform controller main.tf file (see resources below). A new deployment workspace is created in terraform cloud to host the config designer main.tf that will actually deploy the compute and networking modules in the terraform cloud org to AWS.  Note that the oAuth permits this new deployment workspace on terraform cloud to communicate with github to get the deployment main.tf config designer file that deploys the compute and networking terraform modules.  This setup is very highly scalable since it is fully modularized and the newest github VC provider (see below) is added at the org level. The various workspaces in terraform cloud are in the org and have full access to all terraform modules in that org to run in these various workspaces. 

This controller main.tf stays local on the Cloud9 IDE and is not deployed to github or terraform cloud.  This file is terraform applied from an mtc-control directory in this
terraform project workspace on Cloud9 IDE as part of this development project.  Note that the github repo created below is destroyed on terraform destroy. This README file pertains to the deployment flows that the config designer main.tf file on it deploys to AWS. The config designer main.tf file is below this mtc control main.tf file that is right below.

# This is mtc-control/main.tf.  Terraform has been intialized in this directory and can be run locally in Cloud9 IDE


# add a locals block
locals {
    aws_creds = {
        AWS_ACCESS_KEY_ID = var.aws_access_key_id
        AWS_SECRET_ACCESS_KEY = var.aws_secret_access_key
        # note that the CAPS are only used here for the aws_creds.  The local.aws_creds is what is passed to the terraform cloud runners via
        # the tfe provider in providers.tf via the resource "tfe_variable" "aws_creds"  below.
    }
    
    organization = "course7_terraform_adv_AWS_org"
    # my terraform cloud org is the above.
}

# https://registry.terraform.io/providers/integrations/github/latest/docs
# We will use the github provider to create the github deployment repo on github with the /mtc-control/deployments/mtc-dev/main.tf 
# terraform file that will eventually get pushed to a deploymetn workspace on terraform cloud!
# This main.tf will deploy the compute and networking mmodules on terraform cloud to AWS

resource "github_repository" "mtc_repo" {
    name = "mtc-dev-repo"
    description  = "VPC and compute resource deployment repo initiated by terraform"
    auto_init = true
    # it will automatically be initialized on creation
    license_template = "mit"
    
    visibility = "public"
    # I will make mine public.
}

# Next set a default branch
resource "github_branch_default" "default" {
    repository = github_repository.mtc_repo.name
    # this will reference the repo above.
    branch = "main"
}

# Next add the configuration designer main.tf file that was created in terraform cloud. This is in 
# the mtc-control/deployments/mtc-dev/main.tf
resource "github_repository_file" "main_terraform_deployment_file" {
# note cannot use "main.tf" as the name. But that is what this github repo is for, to put main.tf onto it....
    repository = github_repository.mtc_repo.name
    branch = "main"
    file = "main.tf"
    #this is the name of the file as it will be named in the repo and NOT the location of the file
    content = file("./deployments/mtc-dev/main.tf")
    # this main.tf is 2 directories down from the location of this main.tf.  NOTE that this main.tf is to deploy the githhub repo
    # the main.tf that will be placed into this github repo referred to above is the deployment main.tf that will deploy the compute and
    # networking modules from terraform cloud to AWS!!!
    commit_message = "Github deployment repo that is managed by terraform"
    commit_author = "dave mastropolo"
    commit_email = "dave.mastropolo@gmail.com"
    overwrite_on_create = true
    # this will allow overwrites on the main.tf so that we can perform several runs
}


resource "tfe_oauth_client" "mtc_oauth" {
# this oAuth will permit terrafrom cloud to authenticate with github. This is what we did when addubg a VC provider at the org level in 
# terrafrom cloud. We essaentially added authentication between github as the VC provider and terraform cloud
# Here we are doing this strictly through terraform tf file.  This is what permits our deployment workspace on terraform cloud (see next resource below)
# to communicate with github, specifically the deployment github repo that has the composer main.tf which will deploy the compute and networking
# modules that are accessible in the terraform cloud org.
    organization = local.organization
    # this is defined in the locals block at the start of this main.tf file. See above
    api_url = "https://api.github.com"
    http_url = "https://github.com"
    oauth_token = var.github_token
    service_provider = "github"
}

# Next need to attach the github as a client to the terraform cloud workspace (a deployment workspace that will host the main.tf file in 
# mtc-control/deployments/mtc-dev/main.tf)
resource "tfe_workspace" "mtc_workspace" {
    name = github_repository.mtc_repo.name
    # the name create a dependency so that we do not get the terraform cloud workspace being creatd prior to the github repo
    organization = local.organization
    vcs_repo {
        identifier = "${var.github_owner}/${github_repository.mtc_repo.name}"
        # this is basically dmastrop/<the full repo name> = dmastrop/mtc-dev-repo
        oauth_token_id = tfe_oauth_client.mtc_oauth.oauth_token_id
        # this is from the resource right above this
        # we need the _id and not the .id
        # https://registry.terraform.io/providers/hashicorp/tfe/latest/docs
    }
}

# Need env vars so that the terraform cloud runners can authenticate with AWS
resource "tfe_variable" "aws_creds" {
    for_each = local.aws_creds
    # see locals block above
    key = each.key
    value = each.value
    category = "env"
    sensitive = true
    workspace_id = tfe_workspace.mtc_workspace.id
    description = "AWS credentials"
}


This is the terraform config designer main.tf that is put into the new deployment workspace on terraform that is created by the controller main.tf above.
The config designer main.tf is pushed to the github deployment repo "mtc-dev-repo" that is created during terraform apply of the controller main.tf (above) on Cloud9 IDE.
The file is fetched from the local Cloud9 workspace at this location:   content = file("./deployments/mtc-dev/main.tf")
The controller main.tf file above has established an oAuth between this github repo and terraform cloud with the workspace specified above in terraform cloud (a deployment workspace).  So terraform deployment workspace has access to this config designer main.tf. Once the terraform deployment workspace has this config designer main.tf it can then deploy the compute and netwokring modules in the terraform org (and if there are other modules it can do those) to AWS. The AWS authentication between terraform cloud (workspace) and AWS is facilitated by the loca.aws_creds in the main.tf controller file above.   The locals block in the main.tf above has access to the aws access ID and secret access key that is stored in varaibles.tf via terraform.tfvars file locally on the Cloud9 IDE. Note that these files themselves are never pushed to github and thus there is security.


# This is  mtc-control/deployments/mtc-dev/main.tf

# The configuration below was created (generated) from the terraform cloud configuration designer as a component of the 
# registry and the compute and networking modules.  This file was auto-created by the configuration designer after adding 
# the compute module and networking module and then supplying the
# required variables.  This file is being added as main.tf in mtc-control/deployments/mtc-dev/ as part of the terraform CI/CD project
# workspace on Cloud9 IDE
# Note the source below: this file will actually be executed on terraform cloud (it will be pushed from Cloud9 IDE to a git deployment repo
# which will have an oAuth link to terraform cloud.). It will be pushed to a deployment workspace on terraform cloud. This deployment workspace
# will execute the networking and compute modules in the terraform cloud (already there) per the terraform code below...


//--------------------------------------------------------------------
// Variables



//--------------------------------------------------------------------
// Modules
module "compute" {
  source  = "app.terraform.io/course7_terraform_adv_AWS_org/compute/aws"
  # the module is in terraform cloud in registry modules for the org above
  version = "1.0.0"

  aws_region = "us-east-1"
  public_key_material = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCMYp+cg+iK2bkYPDSau7F5lCjMwuWNk4VVAtEw2qJGAUC0a9hErMG/dOt1Ct/S+G1jninr9iVfy12vEo8pi3wiusw/+4lVIV5KMrI3psMjRnpQRRB4TxFlzTKQ7bYtftt6f78UyQ9CyvmFnxBFeKTDOcT9mxK0IlvmzOdtEVbDqmm5/DYx6TXT3Il+c+3zMZhZruU6C1b/3/mjKkSbYu4cCflsV07q7v0fz9EmDIi0RtUSPDv32Am8sL0OHfTPzqnWuPgNs4Q/go/tq4Fvq73ijO0Vqu5mF0Z8JYIy3PooyBmlrj9XBDUF+BgJsV3TThD7YRJNqTgfYjKw0ofjJZh3aZSKIW8sPcZFky+dr+3eOQqLLk5Eo292J4Ti5MkRhaFHB+5kiy9gnNILRaOn8kj0R7q1U/OPNpoa6wBbSIx6DLvTWcH3WXcNy5DCEPWK8FGQBTLug31b7ICu7jBiuOVdqckqro4Z2WATad6UuST5THBwwDV542t8N27U0htSuMs= ubuntu@ip-172-31-11-176"
  public_sg = "${module.networking.public_sg}"
  # NOTE: when adding these variables in configuration designer in terraform cloud, this is why we used interoplation syntax with ${} 
  # Otherwise the above would have been put directly in quotes.
  public_subnets = "${module.networking.public_subnets}"
}

module "networking" {
  source  = "app.terraform.io/course7_terraform_adv_AWS_org/networking/aws"
  # the module is in terraform cloud in registry modules for the org above
  version = "1.0.0"

  access_ip = "0.0.0.0/0"
  aws_region = "us-east-1"
}
