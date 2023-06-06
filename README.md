# CLD - LAB06 : Infrastructure-as-code and configuration management - Terraform, Ansible and Gitlab
**Group S : A. David, T. Van Hove**

**Date : 01.06.2023**

**Teacher : Prof. Marcel Graf**

**Assistant : Rémi Poulard**

In this lab, we are going to deploy a website running in the cloud and using Terraform and Ansible, tools that use the principle of infrastructure as code. We'll be using NGINX and Google Cloud technologies.

In the first part of this lab, we're going to provision the cloud infrastructure. In other words, we'll create the necessary resources on the cloud.

In the second part, we'll configure the virtual machine by installing a web server and configuration files. To do this, we'll use Ansible.

In the third part, which is optional, we'll use Terraform as a team. The solution will be to store the state of Terraform in a version control system.

### Table of content

[toc]

# Task 1: Install Terraform

We installed Terraform on our laptop based on Ubuntu.

![](.\figures\Terraform_installation.png)

# Task 2: Create a cloud infrastructure on Google Compute Engine with Terraform

The project-IDs :

- **labgce-388816**  For Anthony account
- **labgce-388413** For Tim account



Firs, we added the variables values in the terraform.tfvar file. Then we initialized terraform:

![](figures/Task2_Terraform_Init.png)

Then, we created a terraform plan:

![](figures/Task2_terraform_plan.png)



Finally, we validated the plan:

![](figures/Task2_terraform_validate.png)



> Explain the usage of each provided file and its contents by directly  adding comments in the file as needed (we must ensure that you  understood what you have done). In the file `variables.tf` fill the missing documentation parts and link to the online documentation. Copy the modified files to the report.



backend.tf:

```hcl
terraform {
  backend "local" {
    # Local backend stores state files on the local filesystem
    # Adjust the backend configuration as needed (e.g., for remote state storage)
  }
}
```



main.tf:

```hcl
# Defines the provider for Google Cloud Platform
provider "google" {
  project     = var.gcp_project_id  # The GCP project ID
  region      = "europe-west6-a"    # The desired region for resources
  credentials = file("${var.gcp_service_account_key_file_path}")  # Path to GCP service account key file
}

# Resource: Google Compute Engine instance
resource "google_compute_instance" "default" {
  name         = var.gce_instance_name  # Name of the GCE instance
  machine_type = "f1-micro"             # Machine type for the instance
  zone         = "europe-west6-a"       # Zone for the instance

  metadata = {
    # Set the SSH key for the instance
    ssh-keys = "${var.gce_instance_user}:${file("${var.gce_ssh_pub_key_file_path}")}"
  }
  
  # Boot disk image for the instance
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  # Network for the instance
  network_interface {
    network = "default"

    access_config {
      # Include this section to give the VM an external IP address
    }
  }
}

# Resource: Google Compute Engine firewall for SSH
resource "google_compute_firewall" "ssh" {
  name          = "allow-ssh"   # Name of the firewall rule
  network       = "default"     # Network for the firewall rule
  source_ranges = ["0.0.0.0/0"] # Source IP ranges allowed to access

  allow {
    ports    = ["22"]   # Allowed SSH port
    protocol = "tcp"   # Protocol for the firewall rule
  }
}

# Resource: Google Compute Engine firewall for HTTP
resource "google_compute_firewall" "http" {
  name          = "allow-http"   # Name of the firewall rule
  network       = "default"      # Network for the firewall rule
  source_ranges = ["0.0.0.0/0"]  # Source IP ranges allowed to access

  allow {
    ports    = ["80"]   # Allowed HTTP port
    protocol = "tcp"   # Protocol for the firewall rule
  }
}
```



outputs.ft:

```hcl
# Output: GCE instance IP address
output "gce_instance_ip" {
  value = google_compute_instance.default.network_interface.0.access_config.0.nat_ip
}
```



variables.tf:

```hcl
variable "gcp_project_id" {
  description = "The ID of the Google Cloud Platform project"
  type        = string
  nullable    = false
  # Documentation: https://developer.hashicorp.com/terraform/language/values/variables
}

variable "gcp_service_account_key_file_path" {
  description = "The file path to the GCP service account key file"
  type        = string
  nullable    = false
  # Documentation: https://developer.hashicorp.com/terraform/language/values/variables
}

variable "gce_instance_name" {
  description = "The name of the Google Compute Engine instance"
  type        = string
  nullable    = false
  # Documentation: https://developer.hashicorp.com/terraform/language/values/variables
}

variable "gce_instance_user" {
  description = "The username for SSH access to the GCE instance"
  type        = string
  nullable    = false
  # Documentation: https://developer.hashicorp.com/terraform/language/values/variables
}

variable "gce_ssh_pub_key_file_path" {
  description = "The file path to the SSH public key file"
  type        = string
  nullable    = false
  # Documentation: https://developer.hashicorp.com/terraform/language/values/variables
}

```



terraform.tfvars:

```hcl
gcp_project_id = "labgce-388413"
gcp_service_account_key_file_path = "../credentials/labgce-388413-0359dba88e1b.json"
gce_instance_name = "cld-best-instance"
gce_instance_user = "cookie"
gce_ssh_pub_key_file_path = "../credentials/labgce-ssh-key.pub"
```



> Explain what the files created by Terraform are used for.

When we do a `terraform init`, it will create 1 directory and 4 files:

1. `.terraform/`: This directory is created at the root of the Terraform working directory and contains all the necessary files for the Terraform backend and provider plugins.
2. `.terraform.lock.hcl`: This file records the exact versions of the provider plugins used for the configuration. It ensures reproducibility and consistency when working with Terraform.
3. `terraform.tfstate` : If we are using a local backend, Terraform may create this file to store the state of our infrastructure. However, if we are using a remote backend, the state file is typically stored remotely and not created locally.
4. `terraform.tfstate.backup` : If a previous state file exists, Terraform may create a backup of it with this filename. The backup file helps protect against accidental loss or corruption of the state file.



> Where is the Terraform state saved? 

By default, the Terraform state is stored in locally a file name `terraform.state`. 



> Imagine you are working in a team  and the other team members want to use Terraform, too, to manage the  cloud infrastructure. Do you see any problems with this? Explain.

1. When multiple people simultaneously modify the infrastructure, conflicts can appear when committing and merging changes to the Git repository. Since the state file is binary and changes frequently, it can cause conflicts and difficulties in resolving them.
2. Storing the state file in Git requires continuous synchronization between team members. Each team member needs to ensure they have the latest state file before running Terraform commands.
3. The Terraform state file may contain sensitive information, such as resource IDs, credentials, and private IP addresses. Storing it in a Git repository potentially exposes this sensitive information to  unauthorized access.

To solve these problems, we could use a remote backend for storing the Terraform state, such as Terraform Cloud, AWS S3, or Azure Blob Storage. This can provide a centralized and secure storage solution for the state file. It enables better collaboration, concurrency control, and automatic versioning. Each team  member can access the state file without relying on Git synchronization, and sensitive information is protected in a more secure manner.



> What happens if you reapply the configuration (1) without changing `main.tf` (2) with a change in `main.tf`? Do you see any changes in Terraform's output? Why?

(1) If we reapply the configuration whitout making any changes to the `main.tf` file on our Terraform project, it will detect that there are no changes to apply and will simply refresh the state of our infrastructure. This means that it will query the current state of our ressources and update the Terraform status file accordingly, but itl will not make any changes to out infrastructure.

(2) If we make a change to the `maint.tf` file, Terraform will detect the change in the configuration file and compare it to the current state of the infrastructure. Depending on the nature of the change, Terraform may need to modify or recreate resources to align with the updated configuration. The Terraform output will display the changes it plans to make, such as creating new resources, modifying existing resources, or destroying and recreating resources as needed.



> Can you think of examples where Terraform needs to delete parts of the infrastructure to be able to reconfigure it?

Sometimes, Terraform determines that the existing resources do not match the desired configuration, and must takes the necessary steps to reconcile the infrastructure with the new state. For example:

1. If we change the type of a resource in the configuration (e.g., from an EC2 instance to an RDS database), Terraform needs to delete the existing resource and create a new one to run the change.
2. When we modify certain attributes of a resource, such as changing the size or configuration of an instance, Terraform may need to destroy and recreate the resource to apply the changes.
3. If we remove a resource from the configuration, Terraform will plan to delete the corresponding resource from the infrastructure to ensure it aligns with the desired state.
4. If we modify the dependencies between resources, Terraform may need to update the order in which it creates or modifies resources. This can result in deleting and recreating resources to reflect the new dependencies.



> Explain what you would need to do to manage multiple instances.

In our `main.tf` file, we can define multiple resource blocks for each VM instance we want to create. Each resource block represents an individual VM configuration, for example:

```
resource "google_compute_instance" "instance1" {
  # Configuration for instance 1
}

resource "google_compute_instance" "instance2" {
  # Configuration for instance 2
}
```

To create multiple instances more dynamically, we can leverage loops or dynamic expressions. We can use the `count` parameter in resource blocks or the `for_each` parameter to iterate over a list or map of instance configurations. Example with `count`:
```
resource "google_compute_instance" "instance" {
  count = 3

  # Configuration for each instance
}
```

Example with `for_each`:
```
variable "instances" {
  description = "Map of instance configurations"
  type        = map(object({
    # Define instance configuration attributes
  }))
  default = {
    "instance1" = { ... },
    "instance2" = { ... },
  }
}

resource "google_compute_instance" "instance" {
  for_each = var.instances

  # Configuration for each instance
}
```

If we have multiple instances with different configurations, we can  manage variables for each instance by defining them in a separate file or data structure. This allows to specify instance-specific values and easily update the configurations.

As we responded in the previous question, we can set up a remote state backend for improved collaboration and shared state management.

Now, to manage the instances, we can make changes to the configuration (e.g. instance type, security groups, etc.) and apply these changes using `terraform apply`. Terraform will update existing instances in line with the configuration changes.

Finally, if we want to delete specific instances, we have 2 options:

Option 1: Remove Resource Block by simply deleting the  the resource block in our `main.tf` file that corresponds to the instance you want to delete, then run `terraform plan` and `terraform apply`. Terraform will identify that the instance needs to be deleted and handle the removal accordingly.

Option 2: Run `terraform destroy -target=<resource_address>` . This allows us to specifically target and delete a specific resource without modifying the configuration file. This approach is useful when we want to delete a resource without removing its resource block from the configuration permanently.



> Take a screenshot of the Google Cloud Console showing your Google Compute instance and put it in the report.

![](.\figures\google_cloud_instance.png)

# Task 3: Install Ansible

We followed the instructions for installing Ansible and everything went smoothly.

Note that we had to set up a virtual Python environment that we named `ansible-env`.

![](.\figures\ansible_installation.png)



# Task 4: Configure Ansible to connect to the managed VM

Here is the ansible.cfg file content:

```
[defaults]
inventory = hosts
remote_user = cookie
private_key_file = ../credentials/labgce-ssh-key
host_key_checking = false
deprecation_warnings = false
```



Here is the hosts file content:

```
 gce_instance ansible_ssh_host=34.65.3.209
```



Here is a screenshot of the ping and uptime commands:

![](figures/Task3_ping_uptime.png)

# 

> What happens if the  infrastructure is deleted and then recreated with Terraform? What needs  to be updated to access the infrastructure again?

When the infrastructure is recreated, the public IP address of the managed VM might change. We would need to update the `hosts` file with the new public IP address.



# Task 5: Install a web server and configure a web site







> Explain the usage of each file and its contents, add comments to  the different blocks if needed (we must ensure that you understood what  you have done). Link to the online documentation. Link to the online  documentation.







> Copy your hosts file into your report.







# Task 6: Adding a handler for NGINX restart





> Copy the modified playbook into your report.



# Task 7: Test Desired State Configuration principles



> What is the differences between Terraform and Ansible? Can they both achieve the same goal?





> List the advantages and disadvantages of managing your  infrastructure with Terraform/Ansible vs. manually managing your  infrastructure. In which cases is one or the other solution more  suitable?





> Suppose you now have a web server in production that you have  configured using Ansible. You are working in the IT department of a  company and some of your system administrator colleagues who don't use  Ansible have logged manually into some of the servers to fix certain  things. You don't know what they did exactly. What do you need to do to  bring all the server again to the initial state? We'll exclude drastic  changes by your colleagues for this question.



# Task 8 (optionnal) : Configure your infrastructure using a CI/CD Pipeline

 

> Explain the usage of each file and its contents, add comments to  the different blocks if needed (we must ensure that you understood what  you have done). Link to the online documentation.



> Explain what CI/CD stands for.





> Where is the Terraform state saved? Paste a screenshot where you can see the state with its details in GitLab.





>  Paste a link to your GitLab repository as well as a link to a  successful end-to-end pipeline (creation, configuration and destruction  of your infrastructure).



> Why are some steps manual? What do we try to prevent here?



> List the advantages and disadvantages of managing your  infrastructure locally or on a platform such as GitLab. In which cases  is one or the other solution more suitable?
