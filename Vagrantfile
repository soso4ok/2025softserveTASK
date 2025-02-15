# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # Define the Web Server
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/jammy64"
    web.vm.hostname = "web"

    # Networking
    web.vm.network "forwarded_port", guest: 80, host: 8080
    web.vm.network "forwarded_port", guest: 22, host: 2223
    web.vm.network "private_network", ip: "192.168.56.10"

    # Web + Node Exporter installing
    web.vm.provision "shell", inline: <<-SHELL

      sudo apt-get update -y
  
      #  installation nginx for server, mysql for queries
      sudo apt-get install -y nginx mysql-client

      # basic webserver setup
      echo '<h1>Welcome to the Web Server!</h1>' | sudo tee /var/www/html/index.html
      sudo systemctl restart nginx
      sudo systemctl enable nginx

      # installing prometheus worker
      # variables
      NODE_EXPORTER_VERSION="1.8.2"

      cd /tmp

      # downloading and unzipping
      wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
      tar xvfz node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz

      # export in any folder
      sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/bin/
      rm -rf /tmp/node_exporter*

      # creating an user for auto start 
      sudo useradd -rs /bin/false node_exporter
      sudo chown node_exporter:node_exporter /usr/bin/node_exporter
      
      # craeting systemctl procces 
      cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

#applying configuration
      sudo systemctl daemon-reload
      sudo systemctl enable --now node_exporter
    SHELL
  end

# DATABASE
config.vm.define "db" do |db|
  db.vm.box = "ubuntu/jammy64"
  db.vm.hostname = "db"

  # Networking
  db.vm.network "forwarded_port", guest: 22, host: 3333
  db.vm.network "private_network", ip: "192.168.56.11"

  # MySQL + Node Exporter
  db.vm.provision "shell", inline: <<-SHELL

    sudo apt-get update -y

    # database installation
    sudo apt-get install -y mysql-server mysql-client

    # connectivity setup
    sudo sed -i "s/bind-address\\s*=\\s*127.0.0.1/bind-address = 0.0.0.0/" /etc/mysql/mysql.conf.d/mysqld.cnf

    sudo systemctl restart mysql
    sudo systemctl enable mysql


    # creating user `admin` with all permissions
    sudo mysql -uroot -e "CREATE USER IF NOT EXISTS 'admin'@'%' IDENTIFIED BY 'admin';"
    sudo mysql -uroot -e "GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;"
    sudo mysql -uroot -e "FLUSH PRIVILEGES;"

    # database crating
    sudo mysql -uroot -e "CREATE DATABASE IF NOT EXISTS test_db;"

    # table creating
    sudo mysql -uroot -e "USE test_db; CREATE TABLE IF NOT EXISTS users (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      email VARCHAR(100) UNIQUE NOT NULL
    );"

    # crating users
    sudo mysql -uroot -e "USE test_db; INSERT INTO users (name, email) VALUES
    ('Alice Smith', 'alice@example.com'),
    ('Bob Johnson', 'bob@example.com'),
    ('Charlie Brown', 'charlie@example.com');"


    # the same installation proccess of worker as web service
    NODE_EXPORTER_VERSION="1.8.2"
    cd /tmp
    wget -q https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
    tar -xzf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
    sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/bin/
    rm -rf /tmp/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64*

    sudo useradd -rs /bin/false node_exporter || true
    sudo chown node_exporter:node_exporter /usr/bin/node_exporter

    cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

    sudo systemctl daemon-reload
    sudo systemctl enable --now node_exporter

  SHELL
end



  # PROMETHEUS
  config.vm.define "monitoring" do |mon|
    mon.vm.box = "ubuntu/jammy64"
    mon.vm.hostname = "monitoring"

    # Networking
    mon.vm.network "private_network", ip: "192.168.56.12"
    mon.vm.network "forwarded_port", guest: 9090, host: 9090

    # Prometheus
    mon.vm.provision "shell", inline: <<-SHELL
      set -e  

      # variables

      # version
      PROMETHEUS_VERSION="3.1.0"
      # prometheus sirectory
      PROMETHEUS_DIR="/tmp/prometheus-${PROMETHEUS_VERSION}.linux-amd64"
      # prometheus archive
      PROMETHEUS_ARCHIVE="prometheus-${PROMETHEUS_VERSION}.linux-amd64.tar.gz"
      # prometheus configuration
      PROMETHEUS_CONFIG="/etc/prometheus"
      # prometheus database
      PROMETHEUS_TSDATA="/var/lib/prometheus"
      

      # setup enviroment
      sudo apt-get update -y
      sudo apt-get install -y wget tar


      cd /tmp
      # downloading, unzipping promehteus and moving  
      wget -q https://github.com/prometheus/prometheus/releases/download/v${PROMETHEUS_VERSION}/${PROMETHEUS_ARCHIVE}
      tar -xzf ${PROMETHEUS_ARCHIVE}
      sudo useradd --no-create-home --shell /bin/false prometheus
      sudo mkdir -p ${PROMETHEUS_CONFIG} ${PROMETHEUS_TSDATA}
      sudo chown -R prometheus:prometheus ${PROMETHEUS_CONFIG} ${PROMETHEUS_TSDATA}
      sudo mv ${PROMETHEUS_DIR}/prometheus /usr/local/bin/
      sudo chown prometheus:prometheus /usr/local/bin/prometheus
      

      # configuration setup, connection to workers
      cat <<EOF | sudo tee ${PROMETHEUS_CONFIG}/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "nodes"
    static_configs:
      - targets: ["192.168.56.10:9100", "192.168.56.11:9100"]
EOF
      
      sudo chown prometheus:prometheus ${PROMETHEUS_CONFIG}/prometheus.yml
      
      # setup auto running
      cat <<EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Monitoring System
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=${PROMETHEUS_CONFIG}/prometheus.yml \
  --storage.tsdb.path=${PROMETHEUS_TSDATA}
Restart=always

[Install]
WantedBy=multi-user.target
EOF
      
      sudo systemctl daemon-reload
      sudo systemctl enable --now prometheus
    SHELL
  end
end
