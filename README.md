# Terraform Template - Terraform Infrastructure

> **Disclaimer:** This repository is provided purely as a demonstration of these workflows. You are free to use, modify, and adapt the code as you see fit; however, it is offered as-is with no warranty or support of any kind. Use it at your own risk. This is not production-ready code — it should be reviewed, understood, and rewritten to suit your own environment before any real-world use.

This Terraform configuration uses a three-layer modular architecture to deploy a secure Azure web application infrastructure.

## 📁 Folder Structure

```
terraform/
├── environments/
│   └── dev/
│       ├── main.tf                    # Environment-specific configuration
│       ├── variables.tf               # Environment variables
│       ├── outputs.tf                 # Environment outputs
│       └── terraform.tfvars.example   # Example configuration
├── project/
│   ├── main.tf                        # Project-level orchestration
│   ├── variables.tf                   # Project variables
│   └── outputs.tf                     # Project outputs
└── modules/
    └── azurerm/
        ├── resource_group/            # Resource Group module
```

## 🏗️ Architecture Overview

### Three-Layer Design

1. **Environments Layer** (`environments/dev/`)
   - Terraform and provider version constraints
   - Generates random suffix for resource uniqueness
   - Sets environment-specific configuration
   - Calls the project module

2. **Project Layer** (`project/`)
   - Orchestrates all infrastructure components
   - Builds resource names following naming conventions
   - Calls individual resource modules
   - Manages dependencies between resources

3. **Modules Layer** (`modules/azurerm/`)
   - Reusable, single-purpose resource modules
   - Standardized inputs (name, resource_group_name, location, tags)
   - Consistent outputs (id, name, resource-specific outputs)

### Deployed Resources

- **Resource Group**: Container for all resources

## 🚀 Quick Start

### Prerequisites

- [Terraform](https://www.terraform.io/downloads.html) >= 1.0
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- Active Azure subscription with appropriate permissions

### Deployment Steps

1. **Authenticate with Azure**

```bash
az login
az account set --subscription "<your-subscription-id>"
```

2. **Navigate to Environment Directory**

```bash
cd terraform/environments/dev
```

3. **Configure Variables**

Copy and customize the tfvars file:

```bash
cp terraform.tfvars.example terraform.tfvars
```

**⚠️ IMPORTANT**: Edit `terraform.tfvars` and set secure credentials:


4. **Initialize Terraform**

```bash
terraform init
```

5. **Review the Deployment Plan**

```bash
terraform plan
```

6. **Deploy Infrastructure**

```bash
terraform apply
```

Type `yes` when prompted.


## ⚙️ Configuration

### Key Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `environment_prefix` | Environment name | `dev` |
| `workload` | Workload identifier | `terraform` |
| `location` | Azure region | `eastus` |
| `data_location` | Data residency region | `""` (uses location) |

### Resource Naming Convention

Resources follow: `<type>-<workload>-<environment>-<suffix>`

Examples:
- Resource Group: `rg-terraform-dev-a1b`

## 📤 Outputs

After deployment, these outputs are available:

- `resource_group_name` - Resource group name

View all outputs:

```bash
terraform output
```

## 🔧 Module Usage

Each module follows a consistent pattern:

### Module Inputs
```hcl
module "example" {
  source = "../modules/azurerm/<resource>"
  
  name                = "resource-name"
  resource_group_name = "rg-name"
  location            = "eastus"
  tags                = { Environment = "dev" }
  
  # Resource-specific properties
}
```

### Module Outputs
```hcl
output "id" { value = azurerm_<resource>.main.id }
output "name" { value = azurerm_<resource>.main.name }
# Additional resource-specific outputs
```

## 🔄 CI/CD Pipeline Setup

Both the GitHub Actions workflow (`.github/workflows/main.yml`) and the Azure DevOps pipeline (`.ado/pipelines/main.yml`) share the same general flow:

- **CI**: Runs `fmt`, `init`, `validate`, and `plan` on pull requests targeting `main`, then posts results as a PR comment
- **CD**: Triggered manually on `main`; runs `plan`, waits for approval, then runs `apply`

### Azure Prerequisites (Required for Both)

Complete these steps once regardless of which CI/CD platform you use.

#### 1. Create a Service Principal

```bash
az ad sp create-for-rbac \
  --name "sp-terraform-cicd" \
  --role Contributor \
  --scopes /subscriptions/<your-subscription-id> \
  --sdk-auth
```

Save the output — you will need `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`.

#### 2. Create a Storage Account for Terraform State

```bash
# Create a resource group for state storage
az group create \
  --name rg-terraform-state \
  --location eastus

# Create the storage account (name must be globally unique)
az storage account create \
  --name <your-storage-account-name> \
  --resource-group rg-terraform-state \
  --sku Standard_LRS \
  --allow-blob-public-access false

# Create the state container
az storage container create \
  --name tfstate \
  --account-name <your-storage-account-name>
```

#### 3. Grant the Service Principal Access to the State Storage Account

```bash
SP_OBJECT_ID=$(az ad sp show --id <clientId> --query id -o tsv)

az role assignment create \
  --assignee-object-id $SP_OBJECT_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-terraform-state/providers/Microsoft.Storage/storageAccounts/<your-storage-account-name>
```

#### Required Secret Values

| Secret Name | Description |
|---|---|
| `ARM_CLIENT_ID` | Service principal client ID |
| `ARM_CLIENT_SECRET` | Service principal client secret |
| `ARM_SUBSCRIPTION_ID` | Azure subscription ID |
| `ARM_TENANT_ID` | Azure tenant ID |
| `TF_STATE_STORAGE_ACCOUNT` | Storage account name for Terraform state |
| `TF_STATE_RESOURCE_GROUP` | Resource group containing the state storage account |
| `DEV_LOCATION` | Azure region for the dev environment (e.g. `eastus`) |

---

### GitHub Actions Setup

#### 1. Add Repository Secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions → New repository secret** and add each value from the [Required Secret Values](#required-secret-values) table above.

#### 2. Verify the Workflow File

Ensure `.github/workflows/main.yml` exists in the repository. The workflow will activate automatically on the next pull request or push to `main`.

#### 3. Pipeline Behavior

| Trigger | Behavior |
|---|---|
| Pull request targeting `main` | Runs CI: fmt check, init, validate, plan; posts results as a PR comment |
| Manual (`workflow_dispatch`) with stage `Dev` on `main` | Runs CD: init, plan, apply to dev environment |

> The `dev` GitHub Environment can be configured under **Settings → Environments** to add required reviewers or wait timers before the deploy job runs.

---

### Azure DevOps Setup

#### 1. Install the Terraform Extension

Install the **Terraform** extension from the marketplace into your ADO organization. This provides the `TerraformInstaller@1` task used in the pipeline.

[ms-devlabs.custom-terraform-tasks](https://marketplace.visualstudio.com/items?itemName=ms-devlabs.custom-terraform-tasks)

#### 2. Create a Variable Group

In **Pipelines → Library → + Variable group**, create a group named `terraform-secrets` and add each value from the [Required Secret Values](#required-secret-values) table above. Mark sensitive values as secret.

#### 3. Create the Dev Environment

In **Pipelines → Environments → New environment**, create an environment named `dev`. To enforce manual approval before `apply`:

1. Open the `dev` environment
2. Select **Approvals and checks → +**
3. Add an **Approvals** check and specify the required approvers

> The `ManualValidation` task in the pipeline also adds an inline approval gate before the apply job runs, acting as a second confirmation prompt.

#### 4. Enable the Pipeline to Post PR Comments

The CI stage uses `System.AccessToken` to call the ADO REST API and post plan output as a PR comment. To enable this:

1. Edit the pipeline
2. Select **...** (more options) → **Triggers**
3. Under **YAML** → **Get sources**, check **Allow scripts to access the OAuth token**

Alternatively, add the following to the pipeline job:

```yaml
- job: TerraformPlan
  ...
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

This is already included in the pipeline definition.

#### 5. Create the Pipeline

1. In **Pipelines → New pipeline**, select your repository source
2. Choose **Existing Azure Pipelines YAML file**
3. Set the path to `.ado/pipelines/main.yml`
4. Link the `terraform-secrets` variable group under **Variables → Variable groups**
5. Save and run

#### 6. Pipeline Behavior

| Trigger | Behavior |
|---|---|
| Pull request targeting `main` | Runs CI stage: fmt check, init, validate, plan; posts results as a PR thread comment |
| Manual run with stage `Dev` on `main` | Runs CD stage: plan → manual approval → apply to dev environment |

---

## 🧹 Cleanup

To destroy all resources:

```bash
cd terraform/environments/dev
terraform destroy
```

Type `yes` to confirm. This will remove all resources in the resource group.

## 🐛 Troubleshooting

### Common Issues

**Terraform init fails**
- Verify Terraform version >= 1.0
- Check internet connectivity
- Clear `.terraform` directory and retry


## 📚 Additional Resources

- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
