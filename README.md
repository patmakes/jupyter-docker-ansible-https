# Deploy Jupyter Docker Container with Self-Signed Certificate over Tailscale

This repo is used to instruct the deployment of a VM to host a Jupyter Docker server instance with a self-signed SSL certificate. Once the host is provisioned, the included Ansible Playbook can be used to run the SSL certificate generation script, pull the latest Jupyter Container Image from the Quay repository, and configure the docker container to run the Jupyter Notebook server with the self-signed certificate on startup. This documentation assumes that you have already deployed a Tailscale Mesh VPN and configured an Ansible instance to run the playbook.

For more information on Tailscale and Ansible, see the documentation below:

#tailscale Documentation: https://tailscale.com/kb/
#ansible  Documentation: https://docs.ansible.com/

## Step 1 - Provision Target VM for Jupyter Server

First, provision a VM host for the Jupyter Container. The recommended specs will vary depending on your development goals and number of users on the server. These are recommended specifications:

- 2-4 vCPUs
- 4-16 GB RAM
- 128-256GB Free Disk Space

## Step 2 - Install Tailscale on target VM and enroll VM in network

Once your VM is initialized, use the following command to install Tailscale:

`curl -fsSL https://tailscale.com/install.sh | sh`

Then use the generated link to enroll the VM in your Tailnet.

***Once enrolled in the Tailnet, make sure to add the VM's tailscale IP to your Ansible Hosts file***

## Step 3 - Create a Service account for Ansible Script

Create an additional user that can perform nopassword sudo actions. This user will be used for the ansible script and can be disabled after a successful installation. To simplify the permissions between Host and Container the service account on the Host VM will have the same name `jovyan` as the Jupyter container user:

Add the user:

```
useradd -m jovyan
```

Set User password:

```
passwd jovyan
```

Set up user in Sudoers.d with NOPASSWORD for Ansible automation:

```
echo "jovyan ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/jovyan
```

Once you have established SSH access to the VM for both our administrative and service account, for maximum security it is recommended to enable SSH via keys in a secure vault and disable password authentication in the SSH config.

You can generate an ssh key using `ssh-keygen` and copy it to the VM using `ssh-copy-id`

on systems that do not support `ssh-copy-id`, or if you are working with a vault that leverages `ssh-add -L` this command will work as well (althought you might want to run `ssh-add -L` on its own to get the key name first):

```
ssh <VM USER>@<VM IP> "echo $(ssh-add -L | grep <KEY NAME>) >> ~/.ssh/authorized_keys"
```

After adding the SSH to the VM, test SSH with the key instead of the password to ensure that adding the key was successful.

***To Disable Password Authentication for SSH on Ubuntu Server, uncomment the following lines and make the edits below to the ssh_config, sshd_config, and 50-init-cloud files in /etc/ssh***

**ssh_config**

```
   ...
   PasswordAuthentication no
   ...
   GSSAPIAuthentication no
```

**sshd_config**

```
   ...
   PermitRootLogin no
   ...
   PasswordAuthentication no
   PermitEmptyPasswords no
   ...
   UsePAM no
```

**sshd_config.d/50-cloud-init.conf** 

```
PasswordAuthentication no
```

Now you can restart the ssh service or reboot, and test that you have successfully disabled password authentication. 

***Once you have set up SSH keys for the service account, make sure they are accessible to the Hosts file on the Tailscale Control Node, either by path or .env injection***
## Step 4 - If possible, Snapshot Target VM 

If your environment supports snapshots, it is a good idea at this point to snapshot the VM to roll back to a known good state in the event the installation fails in Step 5.

Consult your hypervisor or system documentation for how to snapshot.

## Step 5 - Update SSL config Files in `ssl-config` Directory

The `gen_root_server_certificates.sh` script located in `ssl-config` is run by the playbook on the target VM when the Jupyter Docker container is installed. This script uses the `[req_distinguished_name]` and `[alt_names]` sections of the `.cnf` to configure the SSL certificate that the Jupyter Notebook uses. Update the following sections in `ca_cert.cnf`, `server_cert.cnf`, and `server_ext.cnf` as necessary:

- `ca_cert.cnf`

```
[req_distinguished_name]
countryName             = TODO
stateOrProvinceName     = TODO
localityName            = TODO
organizationName        = TODO
commonName              = TODO
```

- `server_cert.cnf`

```
[req_distinguished_name]
countryName             = TODO
stateOrProvinceName     = TODO
localityName            = TODO
organizationName        = TODO
commonName              = TODO
```

- `server_ext.cnf`

```
[alt_name]
IP.1 = < TODO: IP of HOST VM>
```

## ## Step 6 - Test Ansible Connection to Target VM and run Playbook

To test the connection to the target VM from the ansible control node run:

*in this case the group name is "jupyter"*

```
ansible -m ping <group name from hosts file>
```

If the connection is successful, run the installation playbook with the following command:

*note that the -vvv option is to maximize verbosity to troubleshoot any playbook failures*

```
ansible-playbook -vvv install-jupyter-base.yml
```
## Step 7 - Test Jupyter Notebook

Navigate to the IP specified in the `server_ext.cnf` file and ensure the Jupyter server is properly accessible:

```
https://<Tailscale IP of VM Host>
```

Ignore any warnings from the self-signed certificate in the browser, or install the certificate generated on the Host VM in your browser's certificate chain.
## Step 8 - Disable the service account created for Wazuh Deployment on the Host VM

to disable the service account created for ansible until needed later, run this command on the Host VM:

```
sudo usermod jovyan -s /sbin/nologin
```