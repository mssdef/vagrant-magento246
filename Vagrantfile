# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "geerlingguy/centos7"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  config.vm.network "public_network", bridge: "en0: Wi-Fi (Wireless)"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
    vb.memory = "8024"
    vb.cpus = 2
  end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: <<-SHELL
    sudo yum update -y

    ### SETUP ENV
    source /vagrant/cfg/auth.values

		BASE_URL='http://192.168.33.10'

    #
    # Install necessary packages
    sudo yum install -y epel-release
    sudo yum install -y vim
    sudo yum install -y git nano wget unzip zip curl httpd 

    sudo yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
    sudo yum-config-manager --enable remi-php81
    sudo yum install -y php php-cli php-fpm php-mysqlnd php-opcache php-gd php-curl php-mbstring php-xml php-pear php-bcmath php-json php-common php-intl php-mysqlnd php-soap php-xml php-xsl php-zip

    # Start and enable Apache and MariaDB
    sudo systemctl start httpd
    sudo systemctl enable httpd

    # Install MySQL
    cp /vagrant_data/cfg/MariaDB.repo /etc/yum.repos.d/MariaDB.repo
    yum clean all
    sudo yum install -y MariaDB-server MariaDB-client MariaDB-shared MariaDB-backup MariaDB-common
    systemctl start mariadb
    systemctl enable mariadb

    mysql -u root <<EOF
CREATE DATABASE magento;
GRANT ALL PRIVILEGES ON magento.* TO 'magento'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF

    # Install composer
    curl -sS https://getcomposer.org/installer | php
    mv -f composer.phar /usr/bin/composer

    # Download and install Magento
    cd /var/www/html
    sudo mkdir /usr/share/httpd/.config
    chmod -R 777 /usr/share/httpd ./
    sudo -u apache composer global config http-basic.repo.magento.com ${ADOBE_AUTH_KEY} ${ADOBE_AUTH_SECRET}
    sudo -u apache composer config --global process-timeout 6000

    # Copy pre-downloaded Magento
    tar xzf /vagrant_data/builds/magento-v246.tgz
    # Download magento repo
    #sudo -u apache composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition ./
    

    # install opensearch/elasticsearch
    wget -q https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.7.0-x86_64.rpm
    sudo rpm --install elasticsearch-8.7.0-x86_64.rpm

    rm -f /etc/elasticsearch/elasticsearch.yml
    cp /vagrant/cfg/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
    sudo systemctl enable elasticsearch
    sudo systemctl start elasticsearch

    ## Details log
    pwd
    ls -la
    echo "Start Magento Configuration"


    ## Blank Magento
    sudo php bin/magento setup:install --base-url=http://192.168.33.10/ --db-host=localhost --db-name=magento --db-user=magento --db-password='password' --admin-firstname=admin --admin-lastname=admin --admin-email=admin@example.com --admin-user=${MAGENTO_ADMIN_USER} --admin-password=${MAGENTO_ADMIN_PASS} --language=en_US --currency=USD --timezone=America/New_York --use-rewrites=1

    echo "ServerName localhost" >> /etc/httpd/conf/httpd.conf
    sed -i "s/AllowOverride None/AllowOverride All/g" /etc/httpd/conf/httpd.conf
    systemctl restart httpd

    # Change memory_limit for PHP
		new_value="2512M"
		line_number=$(grep -n "^memory_limit" /etc/php.ini | cut -d':' -f1)
		sed -i "${line_number}s/.*/memory_limit = ${new_value}/" /etc/php.ini
		echo "PHP: memory_limit has been changed to ${new_value} in /etc/php.ini"

    nice -n20 php bin/magento setup:upgrade; bin/magento setup:di:compile; php bin/magento cron:run; php bin/magento indexer:reset; php bin/magento indexer:reindex; php bin/magento cache:disable full_page; bin/magento cache:clean; chmod -R 777 var/ pub/ generated/; echo > var/log/debug.log; echo > var/log/system.log; echo > var/log/support_report.log; curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_cluster/settings -d '{ "transient": { "cluster.routing.allocation.disk.threshold_enabled": false } }'; curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'; php bin/magento indexer:reindex;

    # Post update
    php bin/magento module:disable Magento_TwoFactorAuth Magento_AdminAdobeImsTwoFactorAuth
    php bin/magento c:f
    chmod -R 777 var/ generated/ pub

    ADMIN_URL=`cat app/etc/env.php |grep admin|awk '{print $3}'| sed "s/'//g"`
		echo "Magento web is running: ${BASE_URL}"
		echo "Admin magento: ${BASE_URL}/${ADMIN_URL}"
    echo "${MAGENTO_ADMIN_USER} / ${MAGENTO_ADMIN_PASS}"

  SHELL
end
