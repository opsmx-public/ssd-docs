Terraform-Automation-for-SSD-Installation-through-Kubernetes Job

Git repo : https://github.com/NaveenBalagouni/ssd-use-terraform.git
Doc: https://docs.google.com/document/d/1KDw9L_zrOS5P45985ywRnohPeinD21xbTZH3Qc2dp5c/edit?tab=t.0

1. Overview
This document explains how OpsMx SSD is deployed on Kubernetes using:
Terraform for infrastructure orchestration
Helm for application deployment
Git as the source of truth for Helm charts
A Kubernetes Job to run Terraform inside the cluster

The flow is fully automated and supports repeatable deployments across environments.
Connect to the Kubernetes cluster using kubeconfig (from file or KUBECONFIG environment variable).
Ensure the target namespace exists and protect it from accidental deletion.
Clone the OpsMx SSD Helm chart from the specified Git repository and branch.
Load Helm configuration (values.yaml) directly from Git, keeping configuration version-controlled.
Install OpsMx SSD using Helm with safe, atomic installation settings.
Enable ingress and configure the SSD UI hostname dynamically per environment.
Trigger a fresh installation when Git repository or branch changes, ensuring Git-driven deployments.
Expose Helm release name and namespace as outputs for validation and CI/CD visibility.
Run entirely inside Kubernetes using a Job, enabling repeatable, GitOps-style installations.

2. Architecture Flow
Kubernetes Job starts a Terraform container
Terraform:
Connects to the target Kubernetes cluster using kubeconfig
Creates (or detects) the namespace
Clones the OpsMx SSD Helm chart from Git
Reads Helm values from the repository
Installs  SSD using Helm
Outputs Helm release name and namespace
Directory Structure
Your Terraform project should look like this:
terraform/
├─ main.tf          # Main Terraform configuration
├─ variables.tf     # Variables for customization
├─ outputs.tf       # Outputs for confirmation
├─ terraform.tfvars # Environment-specific values

3. Terraform Providers Configuration
Purpose
Defines required providers and their versions to ensure compatibility and reproducibility.
Tested main.tf 
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "2.16.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

provider "kubernetes" {
  config_path = var.kubeconfig_path != "" ? var.kubeconfig_path : null
}

provider "helm" {
  kubernetes {
    config_path = var.kubeconfig_path != "" ? var.kubeconfig_path : null
  }
}

# Detect or create namespace
# -----------------------------
data "kubernetes_namespace" "opmsx_ns" {
  metadata {
    name = var.namespace
  }
}

# Step 1: Create namespace
resource "kubernetes_namespace" "opmsx_ns" {
  metadata {
    name = var.namespace
  }

  lifecycle {
    prevent_destroy = true
  }
}


# -----------------------------
# Step 1: Clone Helm Chart Repo
# -----------------------------
resource "null_resource" "clone_ssd_chart" {
  triggers = {
    git_repo   = var.git_repo_url
    git_branch = var.git_branch
  }

  provisioner "local-exec" {
    command = <<EOT
      rm -rf /tmp/enterprise-ssd
      git clone --branch ${var.git_branch} ${var.git_repo_url} /tmp/enterprise-ssd
      ls -l /tmp/enterprise-ssd/charts/ssd
    EOT
  }
}

# -----------------------------
# Step 2: Read values.yaml
# -----------------------------
data "local_file" "ssd_values" {
  filename   = "/tmp/enterprise-ssd/charts/ssd/ssd-minimal-values.yaml"
  depends_on = [null_resource.clone_ssd_chart]
}

# -----------------------------
# Step 3: Deploy SSD Helm Releases
# -----------------------------
resource "helm_release" "opsmx_ssd" {
  for_each = toset(var.ingress_hosts)

  depends_on = [null_resource.clone_ssd_chart]

  name      = "ssd-terraform-use"
  namespace = var.namespace
  chart     = "/tmp/enterprise-ssd/charts/ssd"
  values    = [data.local_file.ssd_values.content]
  version    = "2025.04.00"
  
  

  set {
    name  = "ingress.enabled"
    value = "true"
  }

  set {
    name  = "global.certManager.installed"
    value = true
  }

  set {
    name  = "global.ssdUI.host"
    value = each.key
  }

  create_namespace = false
  force_update     = true
  atomic       = true
  recreate_pods    = true
  cleanup_on_fail  = true
  wait             = true

  lifecycle {
    # Combine all attributes you want to ignore in a single ignore_changes list
    ignore_changes = [values, chart, version]
    replace_triggered_by = [null_resource.clone_ssd_chart]
    
  }
}



Providers Used
helm: Deploys Helm charts
kubernetes: Manages Kubernetes resources
local: Reads local files
null: Executes shell commands (Git clone)
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "2.16.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

4. Kubernetes and Helm Provider Setup
Purpose
Allows Terraform to authenticate to Kubernetes using a kubeconfig file.
provider "kubernetes" {
  config_path = var.kubeconfig_path != "" ? var.kubeconfig_path : null
}

provider "helm" {
  kubernetes {
    config_path = var.kubeconfig_path != "" ? var.kubeconfig_path : null
  }
}
If kubeconfig_path is empty, Terraform relies on the KUBECONFIG environment variable (used inside the Job).

5. Namespace Management
Detect Existing Namespace
Terraform first checks if the namespace already exists.
data "kubernetes_namespace" "opmsx_ns" {
  metadata {
    name = var.namespace
  }
}
Create Namespace (If Needed)
Ensures the namespace exists and prevents accidental deletion.
resource "kubernetes_namespace" "opmsx_ns" {
  metadata {
    name = var.namespace
  }

  lifecycle {
    prevent_destroy = true
  }
}

6. Cloning the Helm Chart Repository
Purpose
Fetches the OpsMx SSD Helm chart from Git at runtime.
resource "null_resource" "clone_ssd_chart" {
  triggers = {
    git_repo   = var.git_repo_url
    git_branch = var.git_branch
  }

  provisioner "local-exec" {
    command = <<EOT
      rm -rf /tmp/enterprise-ssd
      git clone --branch ${var.git_branch} ${var.git_repo_url} /tmp/enterprise-ssd
      ls -l /tmp/enterprise-ssd/charts/ssd
    EOT
  }
}
Key Notes
Runs inside the Terraform container
Re-runs automatically if repo URL or branch changes
Uses /tmp as a working directory

7. Reading Helm Values File
Purpose
Loads ssd-minimal-values.yaml from the cloned repository.
data "local_file" "ssd_values" {
  filename   = "/tmp/enterprise-ssd/charts/ssd/ssd-minimal-values.yaml"
  depends_on = [null_resource.clone_ssd_chart]
}
This allows you to:
Keep configuration in Git
Avoid embedding large YAML inside Terraform

8. Deploying OpsMx SSD Using Helm
Helm Release Resource
resource "helm_release" "opsmx_ssd" {
  for_each = toset(var.ingress_hosts)

  name      = "ssd-terraform-use"
  namespace = var.namespace
  chart     = "/tmp/enterprise-ssd/charts/ssd"
  values    = [data.local_file.ssd_values.content]
  version   = "2025.04.00"
Why for_each?
Supports multiple ingress hostnames, creating one Helm release per host.

Helm Overrides (set blocks)
set {
  name  = "ingress.enabled"
  value = "true"
}

set {
  name  = "global.certManager.installed"
  value = true
}

set {
  name  = "global.ssdUI.host"
  value = each.key
}
These override Helm values dynamically per deployment.

Helm Safety and Reliability Flags
atomic          = true
force_update    = true
recreate_pods   = true
cleanup_on_fail = true
wait            = true

Flag
Purpose
atomic
Rollback on failure
force_update
Force resource updates
recreate_pods
Restart pods on change
cleanup_on_fail
Cleanup failed installs
wait
Wait until all resources are ready


Lifecycle Rules
lifecycle {
  ignore_changes = [values, chart, version]
  replace_triggered_by = [null_resource.clone_ssd_chart]
}
Prevents unnecessary redeployments unless Git repo content changes.

9. Outputs
Tested outputs.tf 
output "helm_release_names" {
  value = { for host, r in helm_release.opsmx_ssd : host => r.name }
}

output "helm_namespaces" {
  value = { for host, r in helm_release.opsmx_ssd : host => r.namespace }
}

Helm Release Names
output "helm_release_names" {
  value = { for host, r in helm_release.opsmx_ssd : host => r.name }
}
Helm Namespaces
output "helm_namespaces" {
  value = { for host, r in helm_release.opsmx_ssd : host => r.namespace }
}
Useful for validation and CI/CD visibility.

10. Variables Definition
Tested variables.tf 
variable "git_repo_url" {
  description = "URL of the Git repository containing the OpsMx SSD Helm chart"
  type        = string
  default     = "https://github.com/OpsMx/enterprise-ssd.git"
}

variable "git_branch" {
  description = "Git branch to clone (can be updated for upgrades)"
  type        = string
  default     = "2025-05"
}

variable "kubeconfig_path" {
  description = "Path to your kubeconfig file"
  type        = string
  default     = ""
}


# Ingress Configuration
# ---------------------------------------------
variable "ingress_hosts" {
  description = "The DNS hostname for the SSD UI (must be lowercase)"
  type        = list(string)
}


variable "namespace" {
  description = "Kubernetes namespace to deploy SSD"
  type        = string
  default     = "ssd-tf"
}


variable "helm_release_name" {
  description = "Name of the Helm release"
  type        = string
  default     = "ssd-opsmx-terraform"
}



variable "cert_manager_installed" {
  description = "Set to true if cert-manager is installed"
  type        = bool
  default     = true
}




Key Variables
Variable
Description
git_repo_url
Git repo containing SSD Helm chart
git_branch
Git branch for version control
ingress_hosts
SSD UI DNS names
namespace
Kubernetes namespace
kubeconfig_path
Path to kubeconfig
cert_manager_installed
Enables cert-manager logic


11. terraform.tfvars 
 Tested terraform.tfvars
git_repo_url           = "https://github.com/OpsMx/enterprise-ssd.git"
git_branch             = "2025-05" # initial installation branch
kubeconfig_path        = ""
ingress_hosts          = ["ssd-tf-use.ssd-uat.opsmx.org"]
namespace              = "ssd-opsmx-terraform"
cert_manager_installed = true


12. Kubernetes Job for Terraform Execution
Purpose
Runs Terraform inside Kubernetes, enabling GitOps-style automation.
Job Flow
Clone Terraform repository
Load kubeconfig from secret
Run terraform init
Run terraform plan
Run terraform apply
Tested Job.yaml

apiVersion: batch/v1
kind: Job
metadata:
 name: terraform-job
 namespace: ssd-opsmx-terraform
spec:
 backoffLimit: 1
 template:
   spec:
     serviceAccountName: terraform-sa
     restartPolicy: Never
     containers:
       - name: terraform
         image: hashicorp/terraform:1.6.0
         workingDir: /workspace
         command: ["/bin/sh", "-c"]
         args:
           - |
             set -e


            echo "Cloning Terraform repo..."
            git clone --branch ${GIT_BRANCH} ${GIT_REPO_URL} /workspace/ssd-infra
            cd /workspace/ssd-infra

            echo "Using kubeconfig..."
            export KUBECONFIG=/root/.kube/config

            echo "Terraform init..."
            terraform init

            echo "Terraform plan..."
            terraform plan

            echo "Terraform apply..."
            terraform apply -auto-approve

            echo "Terraform completed successfully"
         env:
           - name: GIT_REPO_URL
             value: "https://github.com/NaveenBalagouni/ssd-use-terraform.git"
           - name: GIT_BRANCH
             value: "main"
         volumeMounts:
           - name: kubeconfig
             mountPath: /root/.kube
             readOnly: true
           - name: workspace
             mountPath: /workspace
     volumes:
       - name: kubeconfig
         secret:
           secretName: kubeconfig
       - name: workspace
         emptyDir: {}




Key Job Configuration
apiVersion: batch/v1
kind: Job
metadata:
  name: terraform-job
  namespace: ssd-opsmx-terraform

Container Execution
image: hashicorp/terraform:1.6.0
workingDir: /workspace
Terraform runs in a clean, repeatable container environment.

Kubeconfig Handling
volumeMounts:
  - name: kubeconfig
    mountPath: /root/.kube
    readOnly: true
Uses a Kubernetes Secret named kubeconfig for secure cluster access.

Commands Executed
terraform init
terraform plan
terraform apply -auto-approve



End of Document


