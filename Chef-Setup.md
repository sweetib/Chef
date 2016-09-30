# Installing and Configuring Chef Server
#### Platform:
* Ubuntu

#### Prerequiste 

* The server you wish to configure as Chef-Server make sure it is accessible by hostname. 

Login into the server, the first task you need to perform is to ensure that the hostname of the server is a resolvable fully qualified domain name (FQDN) or IP address. You can check this by typing:
```sh
hostname -f
```

The result should be an address where the server can be reached. If this is not the case, you can set this to a domain name or IP address where the server can be reached by editing this file:

```sh
cat /etc/hosts
```
 
The file will look similar to this:
```sh
127.0.1.1 current_hostname current_hostname_alias 
127.0.0.1 localhost
```

Modify the top line to reflect the fully qualified domain name or the IP address, followed by a space and any alias you want to use for your host. Add a line beneath the two lines shown that has your server's public IP address in the first column, and the information that you modified at the end of the 127.0.1.1 line to the end. It should look something like this:
```sh
127.0.1.1 fqdn_or_IP_address host_alias
127.0.0.1 localhost
IP_address fqdn_or_IP_address host_alias
```

So, if I do not have a domain name, my IP address is 192.168.0.121, and if I also want my host reachable by the hostname "chef", I could have a file that looks like this:
```sh
127.0.0.1 localhost
127.0.1.1 192.168.0.121 chef.cldcvr.com cldcvr-chef
192.168.0.121 chef.cldcvr.com cldcvr-chef
# The following lines are desirable for IPv6 capable hosts
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

```sh
hostname -f
```

The result should be a value that you can use to reach your Chef server from anywhere in your infrastructure.

#### Download and Install the Chef 12 Server software
```sh
cd ~
wget https://packages.chef.io/stable/ubuntu/14.04/chef-server-core_12.6.0-1_amd64.deb
Once the download is complete, install the package by typing
dpkg -i chef-server-core_12.6.0-1_amd64.deb
```
This will install the base Chef 12 system onto the server. If you have selected a server with less powerful hardware than the recommended amount, this step may fail.
Once the installation is complete, you must call the reconfigure command, which configures the components that make up the server to work together in your specific environment:
```sh
chef-server-ctl reconfigure
```

#### Create an Admin User and Organization

Next, we need to create an admin user. This will be the username that will have access to make changes to the infrastructure components in the organization we will be creating.

```sh
chef-server-ctl user-create USERNAME FIRST_NAME LAST_NAME EMAIL PASSWORD
```

For our example, we will create a user with the following information:
```sh
Username: ccadmin
First Name: ccadmin
Last Name: ccadmin
Email: ccadmin@cldcvr.com
Password: examplepass
Filename: ccadmin.pem
```

```sh
sudo chef-server-ctl user-create ccadmin ccadmin ccadmin  ccadmin@cldcvr.com password -f ccadmin.pem
```

You should now have a private key called `ccadmin.pem` in your current directory.
Now that you have a user, you can create an organization with the org-create subcommand. An organization is simply a grouping of infrastructure and configuration within Chef. The command has the following general syntax:
 
```sh
chef-server-ctl org-create SHORTNAME LONGNAME --association_user USERNAME
```

We will create an organization with the following qualities:
```sh
Short Name: cldcvr
Long Name: cldcvr
Association User: ccadmin
Filename: cldcvr-validator.pem
```

To create an organization with the above qualities, we will use the following command:

```sh
chef-server-ctl org-create cldcvr "cldcvr" --association_user ccadmin -f cldcvr-validator.pem
```

Following this, you should have two `.pem` key files in your home directory. In our case, they will be called `ccadmin.pem` and `cldcvr-validator.pem`.

#### Configure GUI for Chef server
```sh
chef-server-ctl install chef-manage 
chef-server-ctl reconfigure
chef-manage-ctl reconfigure
```

We will need to connect to this server and download these keys to our workstation momentarily. For now though, our Chef server installation is complete.

#### Configure a Chef Workstation

Now that our Chef server is up and running, our next course of action is to configure a workstation. The actual infrastructure coordination and configuration does not take place on the Chef server. This work is done on a workstation which then uploads the data to the server to influence the Chef environment.

**Clone the Chef Repo**: 
* The Chef configuration for your infrastructure is maintained in a hierarchical file structure known collectively as a Chef repo. 
* The general structure of this can be found in a GitHub repository provided by the Chef team.
* We will use git to clone this repo onto our workstation to work as a basis for our infrastructure's Chef repository.

First, we need to install git through the apt packaging tools. Update your packaging index and install the tool by typing:
```sh
apt-get update
apt-get install git
```
Once you have git installed, you can clone the Chef repository onto your machine. For this guide, we will simply clone it to our home directory:
```sh
cd ~
git clone https://github.com/chef/chef-repo.git
```
 
#### Download and Install the Chef Development Kit
The tool we are interested in at this point is the bundled knife command, which can communicate with and control both the Chef server and any Chef clients.
```sh
cd ~
wget https://packages.chef.io/stable/ubuntu/12.04/chefdk_0.14.25-1_amd64.deb
```

Once the `.deb` package has been downloaded, you can install it by typing:
```sh
dpkg -i chefdk_0.14.25-1_amd64.deb 
```
 
After the installation, you can verify that all of the components are available in their expected location through the new chef command:
```sh
chef verify
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
source ~/.bash_profile
```

Download the Authentication Keys to the Workstation
```sh
mkdir ~/chef-repo/.chef
```
 
Download Keys when Connecting to a Chef Server with Passwords
```sh
scp root@192.168.0.121:/root/ccadmin.pem ~/chef-repo/.chef
scp root@192.168.0.121:/root/cldcvr-validator.pem ~/chef-repo/.chef
#If chef server and chef workstation are configured on same server
cp *.pem chef-repo/.chef/.
```

Once you are back on your local computer, you will need to add the SSH keys you use to connect to the Chef server to an SSH agent. OpenSSH, the standard SSH suite, includes an SSH agent that can be started by typing:
```sh
eval $(ssh-agent)
ssh-add 
ssh -A ccadmin@192.168.0.121
```

Configuring Knife to Manage your Chef Environment
```sh
vim chef-repo/.chef/knife.rb
```
 file should have following content:
```
current_dir = File.dirname(__FILE__)
log_level :info
log_location STDOUT
node_name "ccadmin"
client_key "#{current_dir}/ccadmin.pem"
validation_client_name "cldcvr-validator"
validation_key "#{current_dir}/cldcvr-validator.pem"
chef_server_url "https://192.168.0.121/organizations/cldcvr"
syntax_check_cache_path "#{ENV['HOME']}/.chef/syntaxcache"
cookbook_path ["#{current_dir}/../cookbooks"]
```

When you are finished, save and close the knife.rb file.
Now, we will test the configuration file by trying out a simple knife command. We need to be in our 
```sh
~/chef-repo directory for our configuration file to be read correctly:
cd ~/chef-repo
knife client list
```
 
This first attempt should fail with an error that looks like this:

`ERROR:` SSL Validation failure connecting to host: `server_domain_or_IP - SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed`

`ERROR:` Could not establish a secure connection to the server.
Use `knife ssl check` to troubleshoot your SSL configuration.
If your Chef Server uses a self-signed certificate, you can use
`knife ssl fetch` to make knife trust the server's certificates.

`Original Exception:` OpenSSL::SSL::SSLError: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed

Solve error by following command:
```sh
knife ssl check
mkdir ~/chef-repo/.chef/trusted_certs
cp /var/opt/opscode/nginx/ca/192.168.0.121.crt ~/chef-repo/.chef/trusted_certs/.
knife ssl check
knife client list
cldcvr-validator
```

Setup Knife EC2 on workstation
refer: https://github.com/chef/knife-ec2 
Bootstrapping a New Node with Knife
```sh
knife bootstrap 192.168.0.124 -x ccadmin -A --sudo -N test 
knife client list
cldcvr-validator
test
knife node list
test
```
#### Reference link: 
* https://www.digitalocean.com/community/tutorials/how-to-set-up-a-chef-12-configuration-management-system-on-ubuntu-14-04-servers         
* https://docs.chef.io/install_server.html
