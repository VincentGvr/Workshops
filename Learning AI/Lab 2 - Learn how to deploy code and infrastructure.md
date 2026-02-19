# Before Starting 

- Infrastructure (App Service, Foundry, AI Search, etc.) and application code change at different speeds and for different reasons. Mixing them in one pipeline creates fragile deployments.
- We will implement 2 separate pipelines : Why separate?
  - Infra changes are dangerous (destroying a database is irreversible). You want a human to review terraform plan output before apply.
  - App code changes are safe to automate — a bad deploy just gets rolled back.
  - Infra rarely changes; code changes dozens of times a day. Coupling them means every git push runs a slow terraform plan unnecessarily.
  - Terraform outputs resource names/URLs (App Service URL, Foundry endpoint) into GitHub Actions environment variables via terraform output. The app pipeline reads these to know where to deploy.
 
# What we will do 

## Create Teraform infra files 
### providers.tf
1. Create a folder locally `C:\temp\LearnAI`, and two subfolders `\infra` & `\code`
2. Create a first teraform file `providers.tf` responsible for :
   - list of plugins : called providers, it enables interactions with an infrastructure provider, such as Azure (like a library)
   - remote backend information : Teraform needs to store files in a remote location, like a storage account
   - Authentication methods : OIDC :  Workload Identity Federation (OIDC) — GitHub Actions proves its identity to Azure without storing any secret. No client secrets to rotate.
3. Edit the `providers.tf` file created and add :
   - The provider list :  
```
terraform {
  # Pin the AzureRM provider to the 4.x major version.
  # The ~> constraint means "4.x but not 5.x" — allows patch updates, blocks breaking changes.
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
}
```
   - The remote backend information :
```
backend "azurerm" {
    # OIDC authentication — no access keys, no SAS tokens.
    use_oidc         = true # reads ARM_USE_OIDC env var
    use_azuread_auth = true # use Entra ID (AAD) auth instead of storage key

    # These three values identify WHICH Azure account to connect to.
    # Values are injected via env vars in CI, or `az login` locally.
    # tenant_id       = set via ARM_TENANT_ID env var
    # client_id       = set via ARM_CLIENT_ID env var
    # subscription_id = set via ARM_SUBSCRIPTION_ID env var

    # Where the state file lives — created during bootstrap (Phase 1).
    resource_group_name  = "tfstate-rg"
    storage_account_name = "learnaitfstate" # must be globally unique; update after bootstrap
    container_name       = "tfstate"
    key                  = "learnai.terraform.tfstate" # filename inside the container
  }
}
```
   - The provider configuration (for authentication ) :
```
provider "azurerm" {
  features {
    # Soft-delete for Key Vault secrets is enabled by default.
    # During development, purging on destroy is convenient so you can
    # recreate Key Vaults with the same name quickly.
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
}
```
### variables.tf
1. Create a `variables.tf` file. It is a 
