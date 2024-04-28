---
title : "Terraform"
date : "`r Sys.Date()`"
weight : 402
chapter : false
--- 

## Terraform
### Terraform Cloud
- Terraform Cloud is an application that helps teams use Terraform together. It manages Terraform runs in a consistent and reliable environment, and includes easy access to shared state and secret data, access controls for approving changes to infrastructure, a private registry for sharing Terraform modules, detailed policy controls for governing the contents of Terraform configurations, and more.
- Follow these steps to setup Terraform for local development:
  1. If you don't have a Terraform Cloud account, you can create one [here](https://app.terraform.io/public/signup/account).
  2. Install Terraform CLI to your local machine. You can download the Terraform CLI from the [official website](https://www.terraform.io/downloads.html).
  3. Login to your Terraform Cloud account `terraform login`. Choose an existing organization or create a new one. Create a new workspace, name it `ws01`, and select the version control system you are using.

### Terraform project
Use the Amazon Web Services (AWS) provider to interact with the many resources supported by AWS.
- Prepare the Terraform project:
  1. Create a new directory for the project.
  2. In the project directory, create files for the Terraform configuration:
    - **main.tf**: The main configuration file containing the resources to be created.
    - **data-sources.tf**:  Lookup of resources defined outside of Terraform and provide the attributes found within that resource.
    - **locals.tf**: Local variables.
    - **variables.tf**: Define Runtime variables.
    - **outputs.tf**: Define the output variables to other modules.
    - **terraform.tf**: Interactions with cloud providers.
  3. Define providers and Terraform Cloud configuration in the **terraform.tf**  file:
  ```hcl  
  terraform {
    cloud {
      organization = "yourorganizationname"

      workspaces {
        name = "ws01"
      }
    }

    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = "~> 5.0"
      }
    }
  }

  provider "aws" {
    default_tags {
      tags = {
        Management = "ws01-tf"
      }
    }
  }
  ```
{{% notice note %}}
All deployments you're looking for begins with the name `ws01`
{{% /notice %}}
  4. Initialize the Terraform project:
  ```bash
  terraform init
  ```
