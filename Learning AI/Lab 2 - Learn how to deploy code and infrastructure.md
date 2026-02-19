# Teraform, how it works 

`terraform init` 
`terraform apply`
`terraform destroy`

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
1. Still in infra folder, Create a `variables.tf` file. It plays the role of listing the variables used in the different teraform items. It can contain constrains, descriptions, list of values for each variables.
2. Add corresponding variables for project name, environment, location, ai service skus, model name and versions ... :
```
variable "project_name" {
  type        = string
  description = "Short name for this project. Used as a prefix on all Azure resource names to keep them unique and identifiable. Lowercase letters and numbers only, 3-12 chars."
  default     = "learnai"

  validation {
    condition     = can(regex("^[a-z0-9]{3,12}$", var.project_name))
    error_message = "project_name must be 3-12 lowercase letters/numbers only."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment name. Affects resource naming and SKU choices. 'dev' uses cheaper SKUs; 'prod' uses production-grade SKUs."
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

# --- Azure location ---

variable "location" {
  type        = string
  description = "Azure region where all resources will be created. Not all regions support all AI models. East US and Sweden Central are safe choices for GPT-4o + File Search."
  default     = "eastus"
}

# --- AI / Foundry settings ---

variable "ai_services_sku" {
  type        = string
  description = "SKU for the Azure AI Services resource. 'S0' is the standard paid tier that enables model deployments and the Agent Service."
  default     = "S0"
}

variable "openai_model_name" {
  type        = string
  description = "Name of the OpenAI model to deploy inside the Foundry project. gpt-4o is recommended for RAG agents."
  default     = "gpt-4o"
}

variable "openai_model_version" {
  type        = string
  description = "Version of the model to deploy. Check availability in your chosen region at aka.ms/oai/docs/models."
  default     = "2024-11-20"
}

variable "openai_model_capacity" {
  type        = number
  description = "Tokens-per-minute capacity in thousands (TPM/1000). 10 = 10,000 TPM. Start low in dev. Increase for production."
  default     = 10
}

# --- App Service settings ---

variable "app_service_sku" {
  type        = string
  description = "App Service Plan SKU. B1 (Basic) is cheapest for dev/learning. Use P1v3 or higher for production workloads."
  default     = "B1"
}

# --- Tags (applied to all resources) ---

variable "tags" {
  type        = map(string)
  description = "Azure resource tags applied to every resource. Tags help with cost management, filtering, and governance."
  default = {
    project     = "learn-ai-rag-agent"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

### outputs.tf
