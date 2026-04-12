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

# 6. Check the currently authenticated AWS identity.
# This command returns information about the IAM user or role whose credentials are being used.
aws sts get-caller-identity
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

Terraform Core does not know how to talk to AWS, Google Cloud, or Docker directly. Instead, it relies on **Providers**.

- **Plugins:** Providers are executable binaries that act as an abstraction layer. They translate Terraform’s standard resource requests into API calls that the cloud provider understands.
- **The Bridge:** For every cloud service or platform you want to manage (AWS, Azure, GitHub, Kubernetes), there is a corresponding provider.
- **Up-to-date:** Because providers are separate from the core, they can be updated independently to support new cloud features as soon as they are released.

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

### 3. HCL (HashiCorp Configuration Language) Basics

Terraform uses its own domain-specific language called **HashiCorp Configuration Language (HCL)**. It is designed to be highly readable, declarative, and structured. While Terraform can technically consume JSON, HCL is the industry standard due to its support for comments, simpler syntax, and powerful built-in functions.

#### The Anatomy of HCL

Every Terraform configuration file (ending in `.tf`) is built using a combination of **Blocks**, **Arguments**, and **Expressions**.

- **Blocks:** Blocks are the fundamental containers for other content. A block has a type (e.g., `resource`, `provider`), often followed by zero or more labels, and a body enclosed in curly braces `{}`.
- **Arguments:** Arguments assign a value to a specific name. They reside inside blocks. The syntax is simply `key = value`.
- **Expressions:** Expressions represent a value, either literally or by referencing and combining other values.

```bash
block_type "label_1" "label_2" {
  argument_key = "argument_value" # This is a comment
}
```

#### Core Block Types

To get started, you need to understand the three most important block types used in almost every Terraform project.

1. **The `terraform` Block**

This block configures Terraform's underlying behavior, such as specifying the required version of Terraform itself or defining which providers are required.

```bash
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

2. **The `provider` Block**

Providers are the plugins that Terraform uses to interact with cloud platforms (like AWS). The `provider` block configures the connection details, such as the region or authentication methods.

```bash
provider "aws" {
  region = "ap-southeast-1"
  # Note: Credentials should be configured via AWS CLI, not hardcoded here.
}
```

3. **The `resource` Block**

This is the most critical block. It defines a piece of infrastructure that you want to create, such as a virtual machine, a network configuration, or a database.

- **Label 1 (Resource Type):** Defines what you are creating (e.g., `aws_instance`). This is strictly defined by the provider.
- **Label 2 (Local Name):** An identifier you choose to refer to this resource elsewhere in your Terraform code (e.g., `web_server`). This name has no impact on the actual cloud resource name.

```bash
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t3.micro"

  tags = {
    Name        = "Production-Web-Server"
    Environment = "Production"
  }
}
```

#### Comments and Naming Conventions

Writing maintainable IaC requires discipline in commenting and naming.

- **Comments:** HCL supports three comment styles:
  - `#` Single-line comment (Highly Recommended and industry standard).
  - `//` Single-line comment (Alternative, less common).
  - `/* ... */` Multi-line comment.
- **Naming Conventions (Best Practices):**
  - Always use `snake_case` (lowercase words separated by underscores) for local resource names, variables, and outputs.
  - _Bad:_ `resource "aws_instance" "WebServer" { ... }`
  - _Good:_ `resource "aws_instance" "web_server" { ... }`

#### Code Formatting

You do not need to worry about manual indentation. Terraform provides a built-in command to automatically format your code to HashiCorp's exact standards. Always run `terraform fmt` in your terminal before committing code to version control.

### 4. The Core Terraform Workflow

Terraform relies on a strict, sequential workflow to ensure that infrastructure changes are predictable, safe, and reproducible. Mastering this workflow is the day-to-day reality of an Infrastructure Engineer.

The standard deployment lifecycle consists of four primary stages, supplemented by two code-quality steps.

#### `terraform init` (Initialization)

This is the required first step in any new Terraform project or whenever you clone an existing project from Git.

- **What it does:** It initializes the working directory. Terraform scans your `.tf` files to identify which Cloud Providers (e.g., AWS, GCP) and modules are required. It then downloads the necessary provider binaries from the Terraform Registry into a hidden `.terraform` directory.
- **When to run:**
  - Starting a new project.
  - Adding a new provider block.
  - Changing the backend (where the state is stored).

```bash
terraform init
```

#### Pre-flight Checks: `fmt` & `validate`

Before planning or applying, you should ensure your code is syntactically perfect and follows industry standards.

- `terraform fmt`: Automatically rewrites all Terraform configuration files to a canonical format and style (proper indentation, spacing). This ensures consistency across the team.
- `terraform validate`: Checks the configuration for syntax errors and internal consistency (e.g., referencing a variable that hasn't been declared) without accessing any remote cloud APIs.

```bash
terraform fmt
terraform validate
```

#### `terraform plan` (The Dry Run)

This is arguably the most important command. It acts as a safety net.

- **What it does:** Terraform compares your local code against the State file and the actual Cloud environment. It then generates an execution plan outlining exactly what actions it will take to make the real world match your code.
- **Reading the Diff:** Pay close attention to the prefixes in the output:
  - `+` **(Green):** Resource will be created.
  - `-` **(Red):** Resource will be destroyed.
  - `~` **(Yellow):** Resource will be updated/modified in place.
  - `-/+` **(Red/Green):** Resource must be destroyed and recreated (often because you changed an immutable attribute like an AMI ID).
- **Enterprise Best Practice:** In CI/CD pipelines, always save the plan to a file to ensure the exact plan reviewed is the one applied.

```bash
terraform plan -out=tfplan
```

#### `terraform apply` (Execution)

This command executes the actions proposed in the Terraform plan.

- **What it does:** It reaches out to the Cloud Provider APIs and provisions, updates, or deletes the resources.
- **Safety:** By default, it will show you the plan again and ask for manual confirmation (yes) before proceeding.

```bash
# Apply changes interactively
terraform apply

# Apply a pre-saved plan (Standard for CI/CD - bypasses manual 'yes' prompt)
terraform apply "tfplan"
```

#### `terraform destroy` (Teardown)

This command is used to tear down all infrastructure managed by the current Terraform workspace.

- **What it does:** It looks at your state file and permanently deletes every resource it tracks.
- **Use Case:** Primarily used in development or sandbox environments to clean up resources and save costs when testing is complete. **Use with extreme caution in Production.**

```bash
terraform destroy
```

#### Summary of the Developer Loop

1. Write/Modify HCL code.
2. `terraform fmt` -> `terraform validate`.
3. `terraform plan` (Review the output meticulously).
4. `terraform apply` (Deploy).

### 5. Provisioning Your First Resource (`main.tf`)

Now that we understand the HCL syntax and the core workflow, it is time to deploy real infrastructure. In this lab, we will provision a basic AWS EC2 instance (a virtual server) running Ubuntu Linux.

#### Step 1: Project Initialization

Every Terraform project should reside in its own dedicated directory. Open your terminal and create a new workspace:

```bash
# Create a new directory for our first project
mkdir -p ~/terraform-labs/01-first-server
cd ~/terraform-labs/01-first-server

# Open your editor to create the main configuration file
vim main.tf
```

#### Step 2: Writing the Configuration

In Terraform, `main.tf` is the conventional entry point for a root module. Paste the following HCL code into your `main.tf` file. Read through the comments to understand exactly what each block is doing.

```bash
# 1. The Terraform Settings Block
# Defines the required Terraform version and the AWS provider plugin.
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0" # Use version 6.x of the AWS provider
    }
  }
}

# 2. The Provider Block
# Configures the AWS provider. It tells Terraform to deploy resources in the ap-southeast-1 region.
# It automatically uses the credentials you set up earlier with 'aws configure'.
provider "aws" {
  region = "ap-southeast-1"
}

# 3. The Resource Block
# Instructs Terraform to create an AWS EC2 instance.
resource "aws_instance" "app_server" {
  # This is the Amazon Machine Image (AMI) ID for Ubuntu 22.04 LTS in ap-southeast-1.
  # Note: AMI IDs are region-specific. If you use a different region, this ID will change.
  ami           = "ami-0c7217cdde317cfec"

  # The hardware size of the server. t3.micro is eligible for the AWS Free Tier.
  instance_type = "t3.micro"

  # Tags are crucial for enterprise cost tracking and resource management.
  tags = {
    Name        = "Terraform-First-Instance"
    Environment = "Development"
    ManagedBy   = "Terraform"
  }
}
```

#### Step 3: Executing the Deployment Workflow

Now, apply the strict workflow to deploy this server safely.

```bash
# 1. Initialize the Directory
# You will see output showing Terraform downloading the HashiCorp AWS provider.
terraform init

# 2. Format and Validate
# Ensures your code is clean and syntactically correct before proceeding.
terraform fmt
terraform validate

# 3. Generate the Execution Plan
# Read the output carefully. You should see a green '+' indicating that 
# 1 resource (the 'aws_instance.app_server') will be created.
terraform plan

# 4. Apply the Infrastructure
# Terraform will prompt you to confirm. Type 'yes' and press Enter.
# Wait a few seconds, and you will see an 'Apply complete!' message.
terraform apply
```

**Verification:** You can now log into your AWS Management Console via the browser, navigate to the EC2 Dashboard, and select your region (ap-southeast-1). You will see your new server named **"Terraform-First-Instance"** initializing.

#### Step 4: The Clean-Up (Crucial)

In cloud computing, you pay for what you provision. Since this is just a learning exercise, we must destroy the resource to avoid unnecessary AWS charges.

```bash
# Terraform will show you a plan with a red '-' indicating the server will be deleted.
# Type 'yes' to confirm.
terraform destroy
```

Once you see **"Destroy complete!"**, your AWS environment is clean again. You have successfully completed the full Infrastructure as Code lifecycle!
