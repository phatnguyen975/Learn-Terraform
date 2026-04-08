<div align="center">
  <h1>Terraform Tutorial</h1>
  <small>
    <strong>Author:</strong> Nguyễn Tấn Phát
  </small> <br />
  <sub>April 08, 2026</sub>
</div>

## Table of Contents

1. [Environment Setup & Installation](#environment-setup--installation)
2. [Terraform Fundamentals](#terraform-fundamentals)

## Environment Setup & Installation

To build production-grade infrastructure, you must first establish a robust and reproducible local development environment. This guide assumes you are operating within a **Linux environment** (specifically Ubuntu/Debian or WSL2 on Windows).

We will configure the core toolchain required for modern Infrastructure as Code (IaC) development.

### 1. Installing the Terraform CLI

While Terraform might be available in default OS package managers, those versions are frequently outdated. To ensure access to the latest features, security patches, and optimal compatibility (especially critical for the 1.x.x releases and beyond), we will install Terraform directly from HashiCorp's official APT repository.

Execute the following commands in your terminal:

```sh
# 1. Update your system and install essential prerequisite packages.
# 'gnupg' is required for secure key management.
# 'software-properties-common' is to easily manage independent software vendor repositories.
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

# 2. Download HashiCorp's GPG key and output it directly to your trusted keyring.
wget -O - https://apt.releases.hashicorp.com/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# 3. Add the official HashiCorp repository to your package sources.
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 4. Update the package index with the newly added repository and install Terraform.
sudo apt update && sudo apt install terraform

# 5. Verify the installation and check the current version.
terraform -v
```

> **Note on OpenTofu:** In recent years, due to HashiCorp's licensing changes, the Linux Foundation launched OpenTofu as an open-source, drop-in replacement for Terraform. The core concepts, HCL syntax, and commands in this tutorial are 100% compatible with OpenTofu. If your organization mandates fully open-source tooling, you can install OpenTofu instead using their official instructions.

### 2. Configuring the Cloud Provider CLI (AWS)

Terraform acts as an orchestration engine, but it requires explicit permissions to interact with a Cloud Provider's API. For this tutorial, we will utilize **Amazon Web Services (AWS)**.

To authenticate securely, you must install the AWS Command Line Interface (CLI) version 2.

```sh
# 1. Download the latest AWS CLI v2 binary for Linux x86_64 architecture.
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 2. Extract the downloaded archive.
unzip awscliv2.zip

# 3. Execute the installation script.
sudo ./aws/install

# 4. Verify the installation.
aws --version

# 5. Configure your secure credentials.
# You will be prompted to enter your AWS Access Key ID, Secret Access Key,
# default region (e.g., ap-southeast-1), and output format (json).
aws configure
```

> **Security Best Practice:** Never hardcode your AWS credentials directly inside your Terraform code (`.tf` files). Terraform will automatically detect and utilize the credentials configured by the `aws configure` command (stored in `~/.aws/credentials`).

### 3. IDE Optimization for HCL

Writing Terraform configurations requires an editor capable of understanding HashiCorp Configuration Language (HCL). A properly configured IDE will significantly reduce syntax errors and improve development speed through intelligent auto-completion.

- **Visual Studio Code (via WSL Remote):** Install the official **HashiCorp Terraform** extension. It provides robust syntax highlighting, intelligent code completion, and integrated module documentation.
- **Vim / Neovim:** Ensure you have installed the `terraform-ls` (Terraform Language Server). If you are using a plugin manager like `Mason` or `nvim-lspconfig`, installing the Terraform LSP will provide a seamless, native development experience. Add format-on-save (`terraform fmt`) to your configuration to maintain clean code standards.

## Terraform Fundamentals

### 1. What is Infrastructure as Code (IaC)?

### 2. Terraform Architecture
