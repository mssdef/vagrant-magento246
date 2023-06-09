# v2022-05-01
# Read env file from cfg/config.env
env = {}
File.read("cfg/config.env").split("\n").each do |ef|
  env[ef.split("=")[0]] = ef.split("=")[1]
end

Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true;
  config.hostmanager.manage_host = true;
  config.vm.box = "geerlingguy/centos7"

  ### Config RabbitMQ
  config.vm.define "rmq01" do |rmq01|
    rmq01.vm.network "private_network", ip: env['RABBITMQ_HOST']
    rmq01.vm.hostname = "rmq01"

    rmq01.vm.provision "shell", inline: <<-SHELL
      sudo yum -y install epel-release
      sudo yum -y update

      wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
      sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
      sudo yum -y install erlang socat logrotate

      wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.35/rabbitmq-server-3.8.35-1.el8.noarch.rpm
      sudo rpm --import https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
      sudo rpm -Uvh rabbitmq-server-3.8.35-1.el8.noarch.rpm

      sudo systemctl start rabbitmq-server
      sudo systemctl enable rabbitmq-server
    SHELL
  end

  ### Config Memcached
  config.vm.define "mc01" do |mc01|
    mc01.vm.network "private_network", ip: env['MEMCACHED_HOST']
    mc01.vm.hostname = "mc01"

    mc01.vm.provision "shell", inline: <<-SHELL
      # Install Memcache
      yum install memcached -y

      systemctl start memcached
      systemctl enable memcached
      systemctl status memcached
    SHELL
  end

  ### Config MySQL
  config.vm.define "db01" do |db01|
    db01.vm.network "private_network", ip: env['MYSQL_HOST']
    db01.vm.hostname = "db01"
    db01.vm.network "public_network", bridge: "en0: Wi-Fi (Wireless)"
    db01.vm.synced_folder "../data", "/vagrant_data"

    # NOTE: This will enable public access to the opened port
    # db01.vm.network "forwarded_port", guest: 80, host: 8080
    # via 127.0.0.1 to disable public access
    # db01.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

    db01.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end

    # provisioning shell script. Ansible, Chef, Docker, Puppet
    db01.vm.provision "shell", inline: <<-SHELL
      # Install MySQL
      cp /vagrant_data/cfg/MariaDB.repo /etc/yum.repos.d/MariaDB.repo
      yum clean all
      sudo yum install -y MariaDB-server MariaDB-client MariaDB-shared MariaDB-backup MariaDB-common
      systemctl start mariadb
      systemctl enable mariadb

      mysql -u root <<EOF
CREATE DATABASE magento;
GRANT ALL PRIVILEGES ON magento.* TO 'magento'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
    SHELL
  end

  ### Config Magento
  config.vm.define "web01" do |web01|
    # NOTE: This will enable public access to the opened port
    # config.vm.network "forwarded_port", guest: 80, host: 8080
    # via 127.0.0.1 to disable public access
    # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

    web01.vm.network "private_network", ip: env['MAGENTO_HOST']
    web01.vm.network "public_network", bridge: "en0: Wi-Fi (Wireless)"
    web01.vm.hostname = "web01"
    web01.vm.synced_folder "../data", "/vagrant_data"

    web01.vm.provider "virtualbox" do |vb|
      vb.memory = "6024"
      vb.cpus = 2
    end

    # provisioning shell script. Ansible, Chef, Docker, Puppet
    web01.vm.provision "shell", inline: <<-SHELL
      sudo yum update -y

      ### SETUP ENV
      #
      source /vagrant/cfg/config.env
      MYSQL_IP="db01"
      BASE_URL="http://${MAGENTO_HOST}"

      #
      # Install necessary packages
      sudo yum install -y epel-release
      sudo yum install -y vim mariadb
      sudo yum install -y git nano wget unzip zip curl 
      sudo yum -y install nginx

      sudo yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
      sudo yum-config-manager --enable remi-php81
      sudo yum install -y php php-cli php-fpm php-mysqlnd php-opcache php-gd php-curl php-mbstring php-xml php-pear php-bcmath php-json php-common php-intl php-mysqlnd php-soap php-xml php-xsl php-zip

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
      

      # install opensearch
      #
      sudo curl -SL https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/opensearch-2.x.repo -o /etc/yum.repos.d/opensearch-2.x.repo
      sudo yum clean all
      sudo yum repolist
      sudo yum install opensearch -y
      echo "plugins.security.disabled: true" >> /etc/opensearch/opensearch.yml
      systemctl start opensearch 
      systemctl enable opensearch 
      systemctl status opensearch 

      ## Details log
      pwd
      ls -la
      echo "Start Magento Configuration"

      ## Blank Magento
      sudo php bin/magento setup:install --base-url="${BASE_URL}/" --db-host=${MYSQL_IP} --db-name=magento --db-user=magento --db-password='password' --admin-firstname=admin --admin-lastname=admin --admin-email=admin@example.com --admin-user=${MAGENTO_ADMIN_USER} --admin-password=${MAGENTO_ADMIN_PASS} --language=en_US --currency=USD --timezone=America/New_York --use-rewrites=1 
      mysql -h db01 -u magento -ppassword magento < /vagrant_data/builds/db00.sql

      # Configure Nginx
      rm /etc/nginx/nginx.conf
			cp /vagrant_data/cfg/nginx.conf /etc/nginx/nginx.conf
      nginx -t

      echo "# Start Nginx"
      systemctl start nginx
      systemctl enable nginx
      systemctl status nginx

      # Change memory_limit for PHP
      echo "PHP: memory_limit has been changed in /etc/php.ini"
      rm -rf /etc/php.ini /etc/php-fpm.d/www.conf
      cp /vagrant_data/cfg/php.ini /etc/
      cp /vagrant_data/cfg/www.conf /etc/php-fpm.d/

      mkdir -p /var/lib/php/session/
      mkdir -p /run/php-fpm/
      chown -R apache:apache /run/php-fpm/ /var/lib/php/session/
      chmod -R 777 /var/lib/php/session
      systemctl start php-fpm
      systemctl enable php-fpm
      systemctl status php-fpm
      netstat -pl | grep php-fpm.sock

      ## Compile Magento app
      ## magento reset cache
      systemctl restart opensearch; nice -n20 php bin/magento setup:upgrade; bin/magento setup:di:compile; php bin/magento cron:run; php bin/magento indexer:reset; php bin/magento indexer:reindex; php bin/magento cache:disable full_page; bin/magento cache:clean; chmod -R 777 var/ pub/ generated/; echo > var/log/debug.log; echo > var/log/system.log; echo > var/log/support_report.log; curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_cluster/settings -d '{ "transient": { "cluster.routing.allocation.disk.threshold_enabled": false } }'; curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'; php bin/magento indexer:reindex;

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
end
