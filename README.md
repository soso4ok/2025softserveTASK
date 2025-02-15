# Environment Setup

## Prerequisites
Ensure you have the following installed on your system:
- [Vagrant](https://www.vagrantup.com/downloads)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## Setting Up the Environment
1. Clone the repository (if applicable):
   ```sh
   git clone <repository-url>
   cd <repository-folder>
   ```

2. Start the Vagrant environment:
   ```sh
   vagrant up
   ```
   This will create and provision three virtual machines:
   - **Web Server** (IP: 192.168.56.10)
   - **Database Server** (IP: 192.168.56.11)
   - **Monitoring Server** (IP: 192.168.56.12)

## Accessing the Web Server
Once provisioning is complete, you can access the web server via:
- **Localhost:** [http://localhost:8080](http://localhost:8080)
- **Private IP:** [http://192.168.56.10](http://192.168.56.10)

You should see a welcome message: `Welcome to the Web Server!`

## Querying the Database
1. SSH into the database server:
   ```sh
   vagrant ssh db
   ```
2. Connect to MySQL:
   ```sh
   mysql -u root -p
   ```
   Use the default root password (if prompted) or just press `Enter` if not set.

3. Switch to the `test_db` database:
   ```sql
   USE test_db;
   ```

4. List the users in the `users` table:
   ```sql
   SELECT * FROM users;
   ```

## Verifying Monitoring Metrics
Prometheus is installed on the monitoring server and scrapes Node Exporter metrics from both the Web and Database servers.

1. Access Prometheus from your browser:
   - [http://localhost:9090](http://localhost:9090)
   - [http://192.168.56.12:9090](http://192.168.56.12:9090)

2. Verify that Node Exporter targets are being scraped:
   - Navigate to **Status > Targets** in the Prometheus UI
   - You should see `192.168.56.10:9100` and `192.168.56.11:9100` listed

3. Run sample Prometheus queries:
   - `node_memory_MemAvailable_bytes`
   - `node_cpu_seconds_total`

## Stopping the Environment
To stop the Vagrant environment without destroying the virtual machines:
```sh
vagrant halt
```

To completely remove the virtual machines:
```sh
vagrant destroy -f
```

