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



In Terraform, the`terraform.tfvars` file is used to provide input variables for your infrastructure deployments. It is a file with a specific format that allows you to define and assign values to variables used in your Terraform configuration.



> Where is the Terraform state saved? Imagine you are working in a team  and the other team members want to use Terraform, too, to manage the  cloud infrastructure. Do you see any problems with this? Explain.



By default, the Terraform state is stored in locally a file name `terraform.state`. The problem is that if other people want to work on the projet, they don't have this file. When we work in a team, it is recommended to store the state remotely for collaboration and consistency.



> What happens if you reapply the configuration (1) without changing `main.tf` (2) with a change in `main.tf`? Do you see any changes in Terraform's output? Why? Can you think of  exemples where Terraform needs to delete parts of the infrastructure to  be able to reconfigure it?



(1) If wew reapply the configuration whitout making any changes to the `main.tf` file on our Terraform project, it will detect that there are no changes to apply and will simply refresh the state of our infrastructure. This means that it will query the current state of our ressources and update the Terraform status file accordingly, but itl will not make any changes to out infrastructure.

(2) If we make a change to the `maint.tf` file, and then reapply the configuration, Terraform will compare the new desired state with the current state of our infrastructure. It will determine the changes required to achieve the new desired state, and apply these changes accordingly. Terraform will create, update or remove the resources required to match the new configuration.



> Explain what you would need to do to manage multiple instances.



To manage multiple instances in Terraform, we typically have to follow these steps :

1. Define variables : Define in a Terraform configuration make it flexible et reusable. These variables can include things like instance count, instance type and any other parameters that may vary between instances.
2. Create Resource Definition : Define the resource block in our Terraform configuration file to represent a single instance. Whitin the ressource blocl, you can reference the variables defined in step 1 to make it configurable.
3. Use the `count` or `for_each` meta-arguments: you can use the  `count` or `for_each` meta-arguments to control the number of instances you wish to manage. The `count` argument lets you specify a fixed number of instances, while `for_each` lets you define a map or set of instances.
4. Iterate over instances: If you use `for_each`, iterate over instances using a loop in your Terraform code. This allows you to define unique names and other attributes for each instance.
5. Apply configuration: Run terraform apply to create the desired number of instances based on the defined configuration. Terraform will create the instances and track their status in the Terraform status file.
6. Manage instances: Since Terraform manages instances, you can make changes to the configuration (e.g. instance type, security groups, etc.) and apply these changes using terraform apply. Terraform will update existing instances in line with the configuration changes.
7. Destroy instances: If you no longer need the instances, you can run terraform destroy to delete them. Terraform will take care of dismantling the instances and update the status file accordingly.



> Take a screenshot of the Google Cloud Console showing your Google Compute instance and put it in the report.

![](.\figures\google_cloud_instance.png)

# Task 3: Install Ansible

We followed the instructions for installing Ansible and everything went smoothly.

Note that we had to set up a virtual Python environment that we named `ansible-env`.

![](.\figures\ansible_installation.png)

# Task 4: Configure Ansible to connect to the managed VM



> What happens if the  infrastructure is deleted and then recreated with Terraform? What needs  to be updated to access the infrastructure again?





# Task 5: Install a web server and configure a web site







> Explain the usage of each file and its contents, add comments to  the different blocks if needed (we must ensure that you understood what  you have done). Link to the online documentation. Link to the online  documentation.







> Copy your hosts file into your report.







# Task 6: Adding a handler for NGINX restart





> Copy the modified playbook into your report.



# Task 7: Test Desired State Configuration principles<



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
