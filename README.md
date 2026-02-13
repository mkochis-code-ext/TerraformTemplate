# Terraform Template - Terraform Infrastructure

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
