## Terraform scripts

### Software discovery service

```bash
# Navigate to the terraform/software-discovery directory
cd terraform/software-discovery      

# Preview the changes that Terraform will make using the specified variable file
terraform plan -var-file=../vars-files/software-discovery.tfvars

# Apply the changes automatically without prompting for approval, using the specified variable file
terraform apply --auto-approve -var-file=../vars-files/software-discovery.tfvars

# Display the current state of the infrastructure managed by Terraform
terraform show

# Destroy the infrastructure automatically without prompting for approval, using the specified variable file
terraform destroy --auto-approve -var-file=../vars-files/software-discovery.tfvars
```