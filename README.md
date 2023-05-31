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

We installed the Terraform application in a Ubuntu 22.04 system based on WSL and did not encounter any difficulties. The following screenshot shows the version of terraform we used:

![](.\figures\Terraform_installation.png)

# Task 2: Create a cloud infrastructure on Google Compute Engine with Terraform



The project-IDs :

- **labgce-388319** For Anthony account
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



> Where is the Terraform state saved? Imagine you are working in a team  and the other team members want to use Terraform, too, to manage the  cloud infrastructure. Do you see any problems with this? Explain.



> What happens if you reapply the configuration (1) without changing `main.tf` (2) with a change in `main.tf`? Do you see any changes in Terraform's output? Why? Can you think of  exemples where Terraform needs to delete parts of the infrastructure to  be able to reconfigure it?



> Explain what you would need to do to manage multiple instances.



> Take a screenshot of the Google Cloud Console showing your Google Compute instance and put it in the report.



# Task 3: Install Ansible





# Task 4: Configure Ansible to connect to the managed VM





# Task 5: Install a web server and configure a web site





# Task 6: Adding a handler for NGINX restart





# Task 7: Test Desired State Configuration principles



