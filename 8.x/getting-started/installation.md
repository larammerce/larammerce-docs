# Installation

[[toc]]
Larammerce is an open-source ecommerce platform that helps business owners to focus on their concerns and just have the whole complete solution for their online market.

It's been built on top of [Laravel framework](https://laravel.com) which is open source and free to use.

Larammerce has its stack of technologies and utilities as described below:

**Apache** as a web server, **MySQL** version 5.7 as the main relational database, **Redis** database as the cache layer and session management, and **MongoDB** as the log saving database.

## Requirements
Title | Description
------|-----------------
Operation system | gnu/Linux-based operating system (Centos 7 preferred)
Relational database | MySQL 5.7
Interpreter | Php8
WebServer | Apache2/Nginx
Cache DB | Redis
Logs DB | MongoDB
Mail Server | any SMTP server


**NOTE:** In the following part, we are going to run the operating system commands in an interactive shell, Please note that all the commands are executing by the **root** user, if not you should run them by a sudoer user and prepend **sudo** before every command.

**NOTE:** Installation of MongoDB is not required for deployment.

## OS requirements

As every application has its tools and requirements, Larammerce requires some of them as listed below, to install them follow the instructions:

```bash
yum install jq
```


Direnv is an extension for your shell. It augments existing shells with a new feature that can load and unload environment variables depending on the current directory.

Run the following command to install direnv

```bash
curl -sfL https://direnv.net/install.sh | bash
```

Then add the following line at the end of the ~/.bashrc file:

```bash
eval "$(direnv hook bash)"
```

## Install HTTPD (Apache2 web server)

As you know apache2 is used to serve as a web server for our application.

To install apache2 on Centos you have to run the command below:

```bash
yum install httpd
systemctl enable httpd # Make the os to load httpd on startup.
systemctl start httpd # Start the httpd service in the background.
```

As you know at first **httpd** creates a user named **apache** in the system, but there is no shell access for it by default. so to make this user have access to the system shell do the following:

```bash
vim /etc/passwd # open the passwd file to modify it.
```

To change the default shell and home directory for the **apache** user open the passwd file and apply the following changes:

```bash
apache:x:48:48:Apache:/var/www:/bin/bash # change the /bin/nologin to /bin/bash or any other desired shell. 
```

Then set a password for the apache user

```bash
passwd apache # Enter the desired password twice for this command.
```
password is set and then you have to log in as the Apache user with the following command:

```bash
su - apache
```

Modify the current configurations for the httpd:

```
vim /etc/httpd/conf/httpd.conf
```

Find and replace all the `/var/www/html` with the `/var/www/larammerce/public_html`:

```bash
:%s/\/var\/www\/html/\/var\/www\/larammerce\/public_html/g
```

And after all, restart the httpd daemon:

```bash
systemctl restart httpd
```

## Install Node.js and npm

As you know Larammerce is using the Laravel framework as its core system and inherits all Laravel features. for example, Larammerce uses nodejs to build its resource bundles and minify them.

**NOTE:** Larammerce needs node v16.

In the root user run the following curl command to add the NodeSource yum repository to your system:

```bash
curl -sL https://rpm.nodesource.com/setup_16.x | sudo bash -
```
Once the NodeSource repository is enabled, install Node.js and npm by typing:

```bash
sudo yum install nodejs
``` 

## Install MySQL 5.7

As Larammerce is used for production and stable versions of dependencies are needed to be used, the preferred version of Oracle/Mysql is 5.7.

First, we need to enable MySQL 5.7 community to release the "yum" repository on the system.

As the root user run the following command:

```bash
yum localinstall https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
```

As you have successfully enabled MySQL yum repository on your system. Now, install MySQL 5.7 community server using the following commands as per your operating system version.

```bash
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022 #loding the key
yum install mysql-community-server
```
The above command will install the MySQL community server with other dependencies on your system. During the installation process of packages, a temporary password is created and is placed in MySQL log files. Use the following command to find your temporary MySQL password.

First, start the  MySQL Service and enable it, and then run the following command to get the temporary root password:

```bash
systemctl enable mysqld
systemctl start mysqld 
grep 'A temporary password' /var/log/mysqld.log |tail -1 
```

Sample output:

```bash
2017-03-30T02:57:10.981502Z 1 [Note] A temporary password is generated for root@localhost: Nm(!pKkkjo68e
```

Now change the password by running the following command:

```bash
mysql_secure_installation # After running this command the process will begin, enter the copied password, and set the desired configurations according to demands.
```

## Install Redis

Larammerce project uses Redis for some sections, for example, its queue management system, cache server, and session storage.

As the root user run the following command to install the Redis package:

```bash
yum install redis
systemctl enable redis
systemctl start redis
```

## Install php

As you know Centos is based on the stable version of packages, so there is no "php8" on its default repositories.
To install php8 you have to add the **Remi** repo to your list of repositories.

Add the yum repo by running the following command:

```bash
rpm -Uvh https://rpms.remirepo.net/enterprise/remi-release-7.rpm
```
Open/Modify the remi-php80.repo file and edit this repository

```bash
cd /etc/yum.repos.d
vim remi-php80.repo # In the file change the value of the `enable` variable to enable = 1 
```

Then update the list of repositories and existing packages:

```bash
yum update
```

Then install PHP with the following command:

```bash
yum install php
``` 

To use PHP for Larammerce installing some PHP extensions is necessary install them by the following command:

```bash
yum install php-bcmath php-mysql php-pdo php-mbstring php-curl php-imagick php-json php-simplexml php-soap php-xml php-redis php-mongodb php-gd php-zip
```

## Install MongoDB 

By default, MongoDB is used as the logs 'DB' for the Larammerce system.

Create/Modify the following file `/etc/yum.repos.d/mongodb-org-4.4.repo`

```bash
vim /etc/yum.repos.d/mongodb-org-4.4.repo
```
Put the following in the file and then save it:

```repo
[mongodb-org-4.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
```
To install the latest stable version of MongoDB, issue the following command:

```bash
yum install -y mongodb-org
```
You can start the mongod process by issuing the following command:

```bash
systemctl enable mongod
systemctl start mongod
```

## Install composer

Composer is an application-level package manager for the PHP programming language that provides a standard format for managing dependencies of PHP software and required libraries.

First, in the root user install the PHP CLI (command line interface) package and all other dependencies with:

```bash
yum install php-cli php-zip wget unzip
```
Once PHP CLI is installed, download the composer installer script with:

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
```
The command above will download the composer-setup.php file in the current working directory.

To verify the data integrity of the script compare the script SHA-384 hash with the latest installer hash found on the Composer Public Keys / Signatures page.

The following wget command will download the expected signature of the latest Composer installer from the Composer’s Github page and store it in a variable named **HASH**:

```bash
HASH="$(wget -q -O - https://composer.github.io/installer.sig)"
```
To verify that the installation script is not corrupted run the following command:

```bash
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```
Run the following command to install Composer in the /usr/local/bin directory:

```bash
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## Setup Larammerce project

As you know if we want to serve the project files with the apache2 web server, project files must be owned by the apache user. so you have to create the project as the apache user.

Clone the project: 

```bash
su - apache # To login as the apache user
git clone https://gitlab.larammerce.com/larammerce.git # this makes us the directory named larammerce in the path /var/www which is the home directory for the user apache
```
Enter the Larammerce directory

```bash
cd larammerce
```

Run the following command to install node requirements:

```bash
cd larammerce
npm install
```
Then run this command to build the resource files:

```bash
npm run prod # In addition to prod parameter there are some alternatives like 'dev' and 'watch' which for more information you have to refer to laravel/mix projec docs.
```

Run the following command to install Composer dependencies:

```bash
composer install
```

Then you can create a new database with the following command:

```bash
mysql -u root -p -e "create database larammerce_main"
```

Create/Modify environment variables by the following command:

```bash
cp .env.example .env
```

Open the .env file
```bash
vim .env
```
Then edit the file
```bash
APP_NAME=larammerce

APP_URL=http://192.168.80.2:8080

PROXY_URL=http://192.168.80.2:8080

DB_DATABASE=larammerce
DB_USERNAME=root
DB_PASSWORD=password

LOG_DB_DATABASE=larammerce_log
LOG_DB_USERNAME=root
LOG_DB_PASSWORD=

CACHE_DRIVER=redis
SESSION_DRIVER=redis
```
Run the following command to set your application key to a random string.

```bash
php artisan key:generate
```
Then this command to generate jwt secret:

```bash
php artisan jwt:secret
```

**NOTE:** If there is a MySQL dump file you can load it on the database, or can just migrate the DB to start the project database.

Load dumped data to MySQL by the following command:

```bash
mysql -u root -password larammerce_main < template_project.sql
```
If you have no dump file you can just migrate your databse:

```bash
php artisan migrate
```

The most important thing about the Laravel projects is to put the .htaccess file according to Laravel configurations in the document root of the project, so as there is an example file for this use, you can just copy and modify it:

```bash
cd public_html
cp .htaccess.example .htacces
```

## Setup the project template

According to the Larammerce project structure, the project template and backend are developed dedicated and independent of each other.

Clone the template project:

```bash
cd /var/www/larammerce/data
clone https://github.com/path/to/template-project.git
```
Install npm dependencies after cloning the project

```bash
cd template-project
npm install
```

Then to build the project resource files run:
```bash
npm run prod # to build and minify the project resource files.
npm run watch # to watch for resource file changes and then build and export them after every change.
```

Create a new .envrc file from the .envrc.example file

```bash
cp .envrc.example .envrc
```
Add the following to the file named `.envrc`:

```bash
export ECOMMERCE_BASE_PATH=/var/www/larammerce 
```

Then this command to make the direnv package know this directory as its list of directories:

```bash
direnv allow .
```
And after each change in the template, we enter this command again to deploy the resources to the backend directory:

```bash
./deploy.sh
```