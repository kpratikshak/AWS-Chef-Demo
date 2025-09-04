# AWS-Chef-Demo

## Technologies
Chef-Project-Nginx-NodeJS-MySQL

Here are Chef cookbooks to manage different components of a web application stack on Amazon Linux 2023. We will create three cookbooks: one for setting up Nginx as a web server, one for deploying a Node.js application, and one for setting up MySQL as the database server.

Launch EC2 and Connect via SSH

Launch an Amazon EC2 instance running Amazon Linux 2023.
Connect to the instance using SSH.

Step 1: Update the System

Ensure that all system packages are up-to-date with the latest updates and security patches:
sudo yum update -y

To clone the repository from GitHub, follow these instructions:
Cloning the Repository

Ensure Git is installed on your system. If not, install it using the following command:

For Amazon Linux 2023:

sudo yum install git -y

Clone the repository using Git. Open your terminal and run the following command:

git clone https://github.com/kpratikshak/AWS-Chef-Demo.git

Navigate into the cloned repository directory:

cd chef-project

Step 2: Download and Install Chef Workstation

Chef Workstation is a suite of tools for managing infrastructure and executing Chef recipes.

    Download the Chef Workstation package:

    wget https://packages.chef.io/files/stable/chef-workstation/24.4.1064/el/8/chef-workstation-24.4.1064-1.el8.x86_64.rpm

Install the downloaded package:

sudo rpm -Uvh chef-workstation-24.4.1064-1.el8.x86_64.rpm

Step 3: Verify the Installation

Confirm that Chef Workstation is installed correctly by checking the version:

chef --version

Troubleshooting

If you encounter the error : /opt/chef-workstation/embedded/bin/ruby: error while loading shared libraries: libcrypt.so.1: cannot open shared object file: No such file or directory,
resolve it by installing the necessary compatibility library:

sudo yum install libxcrypt-compat -y


## Step 4: Set Up a Chef Repository

Create a Chef repository:

chef generate repo project
cd project

Cookbook 1: Nginx
Step 1: Generate the Nginx Cookbook

chef generate cookbook cookbooks/nginx
cd cookbooks
cd nginx

Step 2: Edit the default Recipe (nginx/recipes/default.rb)
recipes/default.rb

cd reciepes
sudo nano default.rb

package 'nginx' do
  action :install
end

service 'nginx' do
  action [:enable, :start]
end

OR

package 'nginx' do
  action :install
end

service 'nginx' do
  action [:enable, :start]
end

template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  notifies :reload, 'service[nginx]', :delayed
end

Step 3: Create a Template for Nginx Configuration (nginx/templates/default/nginx.conf.erb)

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        location / {
            proxy_pass http://localhost:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

Run receipe

sudo chef-client --local-mode --runlist recipe[nginx::default]

Step 4: Upload the Nginx Cookbook to the Chef Server

knife cookbook upload nginx

Cookbook 2: Node.js Application
Step 1: Generate the Node.js Application Cookbook

chef generate cookbook myapp

Step 2: Edit the default Recipe (myapp/recipes/default.rb)

package 'git'
package 'nodejs'
package 'npm'

git '/srv/myapp' do
  repository 'https://github.com/your-repo/your-app.git'
  revision 'master'
  action :sync
end

execute 'install_dependencies' do
  command 'npm install'
  cwd '/srv/myapp'
  action :run
end

execute 'start_app' do
  command 'npm start &'
  cwd '/srv/myapp'
  action :run
end

Step 3: Upload the Node.js Application Cookbook to the Chef Server

knife cookbook upload myapp

Cookbook 3: MySQL
Step 1: Generate the MySQL Cookbook

chef generate cookbook mysql

Step 2: Edit the default Recipe (mysql/recipes/default.rb)

package 'mysql-server' do
  action :install
end

service 'mysqld' do
  action [:enable, :start]
end

execute 'create_database' do
  command 'mysql -e "CREATE DATABASE myapp;"'
  not_if 'mysql -e "SHOW DATABASES LIKE \'myapp\';"'
end

Step 3: Upload the MySQL Cookbook to the Chef Server

knife cookbook upload mysql

Running the Cookbooks on a Node
Step 1: Bootstrap the Node

knife bootstrap NODE_IP -x ec2-user -i /path/to/key.pem --sudo --node-name NODE_NAME

Step 2: Assign the Cookbooks to the Node's Run List

knife node run_list add NODE_NAME 'recipe[nginx]'
knife node run_list add NODE_NAME 'recipe[myapp]'
knife node run_list add NODE_NAME 'recipe[mysql]'

Step 3: Run Chef Client on the Node

knife ssh "name:NODE_NAME" "sudo chef-client" -x ec2-user -i /path/to/key.pem

Testing with Test Kitchen
Step 1: Install Test Kitchen

chef gem install kitchen
chef gem install kitchen-vagrant
chef gem install kitchen-inspec

Step 2: Create a .kitchen.yml File:

##Kitchen.yaml
---
driver:
  name: vagrant

provisioner:
  name: chef_zero

platforms:
  - name: amazonlinux-2

suites:
  - name: default
    run_list:
      - recipe[nginx::default]
      - recipe[myapp::default]
      - recipe[mysql::default]
    attributes:

Step 3: Run Test Kitchen

kitchen converge
kitchen verify
kitchen destroy

Integration with Jenkins Pipeline
Step 1: Create a Jenkinsfile

pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo/your-cookbook.git'
            }
        }
        stage('Test') {
            steps {
                sh 'kitchen test'
            }
        }
        stage('Deploy') {
            steps {
                sh 'knife cookbook upload your-cookbook'
                sh 'knife ssh "name:NODE_NAME" "sudo chef-client"'
            }
        }
    }
}

Monitoring and Reporting
Step 1: Install and Configure Chef Automate

Follow the Chef Automate installation guide.
Step 2: Configure Reporting

chef-server-ctl enable-reporting

Step 3: Set Up Node Monitoring
Use tools like InSpec to write compliance profiles and run them on nodes.

This project involves generating and uploading cookbooks for Nginx, a Node.js application, and MySQL. We bootstrap a node, assign the cookbooks to the node's run list, and run the Chef client on the node. We also set up testing with Test Kitchen and integrate the deployment process with Jenkins for CI/CD. Finally, we enable reporting and monitoring with Chef Automate. This setup ensures that your web application stack on Amazon Linux 2023 is automated, tested, and continuously integrated.
