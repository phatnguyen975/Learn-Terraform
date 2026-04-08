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
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

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

Before diving into Terraform, it is crucial to understand the paradigm it operates within: **Infrastructure as Code (IaC)**.

Historically, cloud infrastructure was provisioned manually. Systems administrators or DevOps engineers would log into a web portal (like the AWS Management Console), click through various menus, and manually configure servers, databases, and networks. This manual process - often jokingly referred to as **"ClickOps"** - is slow, prone to human error, and impossible to replicate consistently.

**Infrastructure as Code (IaC)** is the modern solution to this problem. It is the process of managing and provisioning computing infrastructure and resources through machine-readable definition files (code), rather than physical hardware configuration or interactive configuration tools.

#### Core Principles of IaC

To master Terraform, you must internalize these three foundational principles of IaC:

1. **Declarative vs. Imperative Approach**
   - **The Imperative Approach (How):** You write scripts that specify the exact step-by-step commands needed to achieve a desired state. (e.g., 1. Check if an EC2 instance exists. 2. If not, run the create command. 3. If yes, check its size. 4. If wrong size, resize it.). Tools like Bash scripts or Ansible (partially) lean imperative.
   - **The Declarative Approach (What):** You define the **final desired state** of your infrastructure, and the IaC tool determines the necessary steps to achieve that state. (e.g., I need 3 EC2 instances of type t3.micro). You don't write the logic of how to create them; the tool figures it out. **Terraform is strictly Declarative.**
2. **Idempotency**
   - Idempotency is a mathematical property meaning that an operation can be applied multiple times without changing the result beyond the initial application. In IaC, if your code says "Create a VPC," and you run that code 100 times, Terraform will create the VPC on the first run, and do **absolutely nothing** on the next 99 runs because the actual state already matches the desired state. This makes deployments incredibly safe and predictable.
3. **Immutable Infrastructure**
   - Traditional IT often relies on _mutable_ infrastructure (servers are updated, patched, and modified in place over time), leading to "configuration drift" where every server becomes a unique "snowflake." IaC promotes **Immutable Infrastructure**. If a server needs an update (e.g., a new OS version), you do not log in and run updates. Instead, you update your code, tear down the old server, and provision a brand new one with the updated configuration.

#### The Business Value of IaC

Why do companies require Senior Cloud Architects to implement IaC?

- **Speed and Efficiency:** Deploying complex, multi-tier architectures takes minutes instead of days.
- **Consistency and Standardization:** Eradicates the "it works on my machine" or "it works in dev but not in prod" problems. Staging and Production environments can be exact clones.
- **Version Control and Collaboration:** Because infrastructure is just text files (`.tf`), it can be stored in Git. This allows for code reviews (Pull Requests), rollback capabilities, and a clear audit trail of _who_ changed _what_ and _when_.
- **Disaster Recovery:** If an entire cloud region goes down, you can re-provision your entire infrastructure in another region almost instantly using your codebase.

### 2. Terraform Architecture

To use Terraform effectively at an enterprise scale, you must understand its internal mechanics. Terraform is designed as a modular, plugin-based system consisting of two primary components: **Terraform Core** and **Terraform Providers**.

#### The High-Level Flow

The Terraform workflow generally follows this logic:

1. **Configuration:** You write HCL code defining the desired state.
2. **Core:** Terraform Core parses the code, compares it against the existing **State**, and builds a **Dependency Graph**.
3. **Providers:** Core uses the graph to tell specific **Providers** to make API calls to the Cloud (e.g., AWS) to create/modify resources.

#### Component: Terraform Core

Terraform Core is a statically-linked binary written in the **Go** programming language. It is the "brain" of the operation. Its responsibilities include:

- **Infrastructure Management:** Reading your configuration files (`.tf`).
- **Resource Graphing:** Identifying dependencies between resources (e.g., a server cannot be created until the network/VPC it sits in exists).
- **State Management:** Maintaining a database of what has been deployed.
- **Execution Planning:** Calculating the difference (the "diff") between the current state and the desired state.

#### Component: Terraform Providers

Terraform Core does not know how to talk to AWS, Google Cloud, or Docker directly. Instead, it relies on Providers.

Plugins: Providers are executable binaries that act as an abstraction layer. They translate Terraform’s standard resource requests into API calls that the cloud provider understands.

The Bridge: For every cloud service or platform you want to manage (AWS, Azure, GitHub, Kubernetes), there is a corresponding provider.

Up-to-date: Because providers are separate from the core, they can be updated independently to support new cloud features as soon as they are released.

#### Component: The State File (`terraform.tfstate`)

The **State File** is arguably the most critical part of the architecture.

- **The Source of Truth:** It is a JSON file that maps your HCL resources to the real-world IDs in your cloud account.
- **Performance:** Instead of querying the cloud provider for every single resource every time you run a command (which would be extremely slow), Terraform looks at the state file to understand the current environment.
- **Metadata:** It stores sensitive information and metadata that isn't necessarily present in your code but is required for resource management.

#### The Resource Graph

Terraform builds a **Directed Acyclic Graph (DAG)** to model your infrastructure. This allows Terraform to:

- **Parallelize Actions:** If two resources (like two independent S3 buckets) have no dependencies on each other, Terraform will create them simultaneously to save time.
- **Determine Order:** It ensures that foundational resources (like a Network) are always created before dependent resources (like a Database).

#### Key Lifecycle Phases

When you run Terraform, it moves through these logical steps:

1. **Refresh:** Queries the cloud provider to see if any "out-of-band" changes happened (someone manually deleted a server).
2. **Plan:** Determines what needs to be done to reach the state defined in your code.
3. **Apply:** Executes the plan via the Providers.
