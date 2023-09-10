# How to install Apache on Almalinux 8 on VirtualBox

This tutorial will demonstrate how to install Apache on Almalinux.

## Prerequisites

- Linux Operating System
- Install [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads)
- Install [Vagrant](https://developer.hashicorp.com/vagrant/downloads)

## Install Almalinux8

Create a folder to work in.

`mkdir -p vagrant/almalinux`

`cd vagrant/almalinux`

We will use the Almalinux image provided by [Vagrant Cloud](https://app.vagrantup.com/boxes/search). Search for alamalinux 8 in the search box.
Click on the Almalinux OS 8 Official Vagrant boxes.

In the How to use this box with Vagrant, click New and copy the command
`vagrant init almalinux/8` and run it on your terminal. This will create a file named Vagrantfile. We will modify the file to use bash commands to update the virtual machine and install Apache.

Open the Vagrantfile and edit as below:

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "almalinux/8"
  config.vm.box_check_update = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider "virtualbox" do |vb|

    # Customize the amount of memory on the VM:
    vb.memory = "2048"
  end

  config.vm.define "webserver" do |web|
    web.vm.hostname = "llcvhost1"
    web.vm.network "private_network", ip: "192.168.56.10" # allows host to access the VM guest
    web.vm.network "forwarded_port", guest: 80, host: 8080
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo su -
    dnf -y update
    dnf -y install epel-release
    dnf -y install git curl httpd
    systemctl start httpd
    systemctl enable httpd
    systemctl start firewalld && systemctl enable firewalld
    httpd -v
    firewall-cmd --zone=public --add-port=80/tcp
    firewall-cmd --zone=public --add-port=443/tcp
    firewall-cmd --zone=public --permanent --add-port=80/tcp
    firewall-cmd --zone=public --permanent --add-port=443/tcp
    firewall-cmd --reload
    firewall-cmd --zone=public --list-ports
    curl -o default.txt http://localhost:80
    grep -n "Test Page for the HTTP Server on AlmaLinux" default.txt

    ############ APACHE CONFIGURATION ############

    APACHE_CONFIG=/etc/httpd/conf/httpd.conf
    VIRTUAL_HOST=${HOSTNAME}
    DOCUMENT_ROOT=/var/www/html/akan/
    echo -e "--- Adding ServerName to Apache config ---\n"
    grep -q "ServerName ${VIRTUAL_HOST}" "${APACHE_CONFIG}" || echo "ServerName ${VIRTUAL_HOST}" >> "${APACHE_CONFIG}"
    sed -i "s/AllowOverride None/AllowOverride All/g" ${APACHE_CONFIG}

    ##### Clone the web application to deploy from Git #####
    echo "--- Cloning web application from GitHub ---"
    cd /var/www/html/
    git clone https://github.com/tisurai/akan.git

    sudo mkdir -p /var/www/html/akan/log
    sudo chown -R vagrant:vagrant /var/www/html/akan
    sudo chmod -R 755 /var/www/
    mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled
    echo "IncludeOptional sites-enabled/*.conf" >> /etc/httpd/conf/httpd.conf
    # Configure virtualhost file for akan application
    echo "--- Configuring Virtualhost file for akan application ---"
    echo -e "<VirtualHost *:80>\n ServerName llcvhost1\n ServerAlias llcvhost1\n DocumentRoot /var/www/html/akan\n ErrorLog /var/www/html/akan/log/error.log\n CustomLog /var/www/html/akan/log/requests.log combined\n</VirtualHost>\n" | tee /etc/httpd/sites-available/akan.conf

    # Create a symbolic link to the sites-enabled directory
    ln -s /etc/httpd/sites-available/akan.conf /etc/httpd/sites-enabled/akan.conf

    ####### Configure SELinux permissions for VirtualHost ########
    # Set universal privileges for Apache
    setsebool -P httpd_unified 1
    # Set privileges on Apache directories
    ls -dlZ /var/www/html/akan/log/

    systemctl restart httpd

    # Test the virtual host
    ls -lZ /var/www/html/akan/log/
  SHELL

end

```

## Bring up the Virtual Machine

Run the command `vagrant up`. This will create a virtual machine and will run the shell commands.

The default user is vagrant

## Login to the virtual machine

Once the commands are executed you can login to the virtual machine by running the command `vagrant ssh`. Run the following commands:

`ls`

`cat default.txt` # to view the contents of the default Apache page

`vagrant provision` # In case you add a shell command, you can run the provision section

## Access the deployed application

Open your browser and type http://192.168.56.10. The application should now be accessible. Congratulations! You have installed and configured Apache web server and deployed an application using a VirtualHost.

## Conclusion

Vagrant is a DevOps tool that can be used to automate provisioning of virtual machines. A bash shell script has been used to configure Apache and deploy a web application from GitHub. Other popular provisioners like Ansible can be used to automate the installation and configuration of the virtual machines. Ansible is useful for managing a large fleet of servers and has advanced features to support rollback operations.
