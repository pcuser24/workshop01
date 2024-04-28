---
title : "Terraform"
date : "`r Sys.Date()`"
weight : 51
chapter : false
--- 

## Terraform
Destroy the infrastructure created by Terraform.
```bash
terraform destroy
```
Just DO IT! Ya know what I mean.
{{% notice info %}}
If a resouce having lifecycle configuration `prevent_destroy`, Terraform will not delete it. You have to manually delete it.
{{% /notice %}}
