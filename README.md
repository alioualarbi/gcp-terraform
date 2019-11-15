# gcp-basic-terraform
#### installing terraform
```
$ wget https://releases.hashicorp.com/terraform/0.12.15/terraform_0.12.15_linux_amd64.zip
$ unzip terraform_0.12.15_linux_amd64.zip
$ sudo mv terraform /usr/local/bin/
```
You need to modify your profile in your home directory, you may find sometimes differemt name such as  .profile or .bash_profile
```
$ vi .bash_profile
```
Add at th end the line below to export the PATH.

```
export PATH="$PATH:/usr/local/bin"
```
After run source command line
```
$ source ~/.bash_profile
```
To verify the installation run:

```
$ terraform -v
```
For setting Up terraform correctly, we need to set up a service account key, which Terraform will use to create and manage resources in your GCP project. Go to the [create service account key page](https://console.cloud.google.com/apis/credentials/serviceaccountkey?_ga=2.181643812.-2022376364.1572904167). Select the default service account or create a new one, select JSON as the key type, and click Create.

This downloads a JSON file with all the credentials that will be needed for Terraform to manage the resources. This file should be located in a secure place for production projects, but for this example move the downloaded JSON file to the project directory or any location convinient to be used.

Note: make sure you have the right permission for the resources needed to be created, most of the time it assigned ```role/editor```

#### Setting up Terraform
Let's creat our work directory for the project and create a ```main.tf``` file for the Terraform config. The contents of this file describe all of the GCP resources that will be used in the project {in our case it's GCP provider [see more provider](https://www.terraform.io/docs/providers/google/index.html)}.
```
// Configure the Google Cloud provider
provider "google" {
 credentials = "${file("CREDENTIALS_FILE.json")}"
 project     = "Project-ID"
 region      = "us-west1"
}
```
The provider “google” line indicates that we are using the Google Cloud Terraform provider and at this point you can run ```terraform init``` to download the latest version of the provider and build the .terraform directory.

```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Checking for available provider plugins...
- Downloading plugin for provider "google" (hashicorp/google) 2.20.0...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.google: version = "~> 2.20"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```
#### Let's Configure the Compute Engine resource

We will add the google_compute_instance resource to the ```main.tf```:

```
// Terraform plugin for creating random ids
resource "random_id" "instance_id" {
 byte_length = 8
}

// A single Google Cloud Engine instance
resource "google_compute_instance" "default" {
 name         = "gcp-vm-${random_id.instance_id.hex}"
 machine_type = "f1-micro" # you can select the machine type desired
 zone         = "us-west1-a" 

 boot_disk {
   initialize_params {
     image = "debian-cloud/debian-9" # the image used
   }
 }

// Make sure flask is installed on all new instances for later steps by adding a startup-script
 metadata_startup_script = "sudo apt-get update; sudo apt-get install -yq build-essential python-pip rsync; pip install flask"

 network_interface {
   network = "default" # which network will be used

   access_config {
     // Include this section to give the VM an external ip address
   }
 }
}
```
After this we can run ``` terraform plan``` to see what we be created as preview.
```
terraform plan 
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_compute_instance.default will be created
  + resource "google_compute_instance" "default" {
      + can_ip_forward          = false
      + cpu_platform            = (known after apply)
      + deletion_protection     = false
      + guest_accelerator       = (known after apply)
      + id                      = (known after apply)
      + instance_id             = (known after apply)
      + label_fingerprint       = (known after apply)
      + machine_type            = "f1-micro"
      + metadata_fingerprint    = (known after apply)
      + metadata_startup_script = "sudo apt-get update; sudo apt-get install -yq build-essential python-pip rsync; pip install flask"
      + name                    = (known after apply)
      + project                 = (known after apply)
      + self_link               = (known after apply)
      + tags_fingerprint        = (known after apply)
      + zone                    = "us-west1-a"

      + boot_disk {
          + auto_delete                = true
          + device_name                = (known after apply)
          + disk_encryption_key_sha256 = (known after apply)
          + kms_key_self_link          = (known after apply)
          + mode                       = "READ_WRITE"
          + source                     = (known after apply)

          + initialize_params {
              + image  = "debian-cloud/debian-9"
              + labels = (known after apply)
              + size   = (known after apply)
              + type   = (known after apply)
            }
        }

      + network_interface {
          + address            = (known after apply)
          + name               = (known after apply)
          + network            = "default"
          + network_ip         = (known after apply)
          + subnetwork         = (known after apply)
          + subnetwork_project = (known after apply)

          + access_config {
              + assigned_nat_ip = (known after apply)
              + nat_ip          = (known after apply)
              + network_tier    = (known after apply)
            }
        }

      + scheduling {
          + automatic_restart   = (known after apply)
          + on_host_maintenance = (known after apply)
          + preemptible         = (known after apply)

          + node_affinities {
              + key      = (known after apply)
              + operator = (known after apply)
              + values   = (known after apply)
            }
        }
    }

  # random_id.instance_id will be created
  + resource "random_id" "instance_id" {
      + b64         = (known after apply)
      + b64_std     = (known after apply)
      + b64_url     = (known after apply)
      + byte_length = 8
      + dec         = (known after apply)
      + hex         = (known after apply)
      + id          = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.

```
Now it's time to run ```terraform apply``` and Terraform will call GCP APIs to set up the new instance! Check the VM Instances page, and the new instance will be there.

#### Let's Run a web server on the new GCP created

First we need to Add SSH access to the Compute Engine instance to access and manage it. Add the local location of your public key to the google_compute_instace metadata in ```main.tf``` to add your SSH key to the instance.

INSERT_USERNAME must be the ssh USERNAME will be used.
```
resource "google_compute_instance" "default" {
 ...
metadata = {
   ssh-keys = "INSERT_USERNAME:${file("~/.ssh/id_rsa.pub")}"
 }
}
```
#### Use output variables for the IP address
Use a [Terraform output variable](https://learn.hashicorp.com/terraform/getting-started/outputs.html) to act as a helper to expose the instance's ip address. For this add the following to the Terraform config
```
// A variable for extracting the external ip of the instance
output "ip" {
 value = "${google_compute_instance.default.network_interface.0.access_config.0.nat_ip}"
}
```
#### Output of terraform apply
```

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

random_id.instance_id: Creating...
random_id.instance_id: Creation complete after 0s [id=4xLxtNZ4GA0]
google_compute_instance.default: Creating...
google_compute_instance.default: Still creating... [10s elapsed]
google_compute_instance.default: Creation complete after 10s [id=gcp-vm-e312f1b4d678180d]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

ip = XXX.XXX.XXX.XXX
```
Run ```terraform apply``` followed by ```terraform output ip``` to return the instance's external IP address. Validate that everything is set up correctly at this point by connecting to that IP address with SSH
```
ssh `terraform output ip`
```
#### Cleaning up
```
$ terraform destroy
```
