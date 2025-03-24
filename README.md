# recruitment-task-devops-master
-----------------------------

## Intro
- local architecture
	- environment specification
   		- hypervisor: Microsoft Hyper-V
   		- gen. 1 virtual machines deployed with two attached network interfaces - one for internet access, one for accessing services within local private subnet
		- os: Ubuntu 24.04 LTS
		- network interfaces (same config for all VMs within HV)
			- **eth0**: `172.17.80.0/28`, internet access (Default Switch)
			- **eth1**: `172.28.160.0/28`, no internet access (Internal Switch)
		- virtual machines and their exposed ports
			- jenkins-main (`172.28.160.2`)
				- port `8080`
			- jenkins-node (`172.28.160.3`)
			- postgres (`172.28.160.4`)
				- port `5432`
			- redis (`172.28.160.5`)
				- port `6379`
			- roadrunner (`172.28.160.6`)
				- port `8080`
     		- all exposed ports are available only via **eth1 interface** (as in `172.28.160.0/28` subnet), the exception being docker container for roadrunner/app project which would normally be exposed either on eth0 interface or it would have iptables pre-routing rule for forwarding incoming traffic to port 8080 (depending on the deployment method)
		- for the sake of simplicity and the fact I'm using Hyper-V homelab, I refrained from setting up HTTPS certificates throughout the Internal Switch network (which would be self-signed anyway). under normal circumstances Jenkins main node would have a dedicated FQDN with a proper Let's Encrypt certificate (preferably issued by certbot)
 
![Local Architecture](https://github.com/user-attachments/assets/e03afc50-b948-4d4e-aa0c-a1df5861e8b8)

		
## Deployment procedure
### basic server configuration (common)
- HV-specific config (post-installation):
	- modify `/etc/netplan/50-cloud-init.yaml` file
	```
 	...
	eth1:
		dhcp4: false
		addresses:
		- 172.28.160.x/28
		routes:
		- to: 172.28.160.0/28
		  via: 172.28.160.1
		nameservers:
			addresses: [172.28.160.1]
	```			
	- `sudo netplan generate`
	- `sudo netplan apply`
	- `sudo reboot`
- set timezone and apply system-wide update
	- `sudo timedatectl set-timezone Europe/Warsaw`
	- `sudo apt update`
	- `sudo apt upgrade -y`
- install utility tools
	- `sudo apt install net-tools`
	- `sudo apt install inetutils-traceroute`
- configure ssh
	- add generated SSH key.pub to `~/.ssh/authorized_keys`
	- modify `/etc/ssh/sshd_config` to disable plain text password login and enforce SSH key authorization
		- `PermitRootLogin no`
		- `ChallengeResponseAuthentication no`
		- `PasswordAuthentication no`
			- (if present) set the same parameter to 'no' in `/etc/ssh/sshd_config.d/50-cloud-init.conf`, or just comment it out
	- `sudo service ssh restart`
		
		
### jenkins-main
- install dependencies and main jenkins package
	- `sudo apt install fontconfig openjdk-17-jre`
	```
	   sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
		  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
	```
	```
	   echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
		  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
		  /etc/apt/sources.list.d/jenkins.list > /dev/null
	```
	- `sudo apt update`
	- `sudo apt install jenkins`
- setup address on which Jenkins will be listening on by default
	- `/usr/lib/systemd/system/jenkins.service`
	```...
	Environment="JENKINS_LISTEN_ADDRESS=172.28.160.2"
	```
	- `sudo systemctl daemon-reload`
- enable service and verify its status
	- `sudo systemctl enable jenkins`
	- `sudo systemctl start jenkins`
	- `systemctl status jenkins`
	- (optional) `sudo netstat -tulpn | grep 8080`, to see if jenkins is in fact listening on default port 8080
- initial configuration
	- retrieve initial admin password from `/var/log/syslog` and login at `http://172.28.160.2:8080`
	- setup custom account with more complex password
	- set worker count to `0` in main node configuration
	
### jenkins-node
- jenkins agent setup
	- install dependencies, setup user/homedir and permissions
		- `sudo apt install --no-install-recommends openjdk-17-jdk-headless`
		- `sudo mkdir /var/lib/jenkins`
		- `sudo groupadd jenkins`
		- `sudo useradd -m -d /var/lib/jenkins -g jenkins -s /bin/bash jenkins`
	- configure agent/node
		- on `jenkins-main` configure new node
			- name: `jenkins-node`
			- description: whatever fits the situation, in this case I just entered VM ip address for future reference
			- number of executors: `2` (as the VM has 2 available vCores)
			- remote filesystem: `/var/lib/jenkins`
			- labels: `linux docker php`
			- usage: `only build jobs with label expressions matching this node`
			- launch method: `launch agent by connecting it to the controller`
			- availability: `keep this agent online as much as possible`
		- as jenkins user, follow instructions available in node summary screen to properly bind agent with main node
			- `echo <secret> > secret-file`
			- `curl -sO http://172.28.160.2:8080/jnlpJars/agent.jar`
			- to test agent out, run `agent.jar` executable directly form the terminal
				- `java -jar agent.jar -url http://172.28.160.2:8080/ -secret @secret-file -name "jenkins-node" -webSocket -workDir "/var/lib/jenkins"`
			- to make it persistent (as in create systemd service)
				- `sudo mkdir -p /usr/local/jenkins-service`
				- `sudo chown jenkins: /usr/local/jenkins-service`
      				- create startup script and allow for its execution
					- `sudo nano /usr/local/jenkins-service/start-agent.sh`
					```
					#!/bin/bash
					cd /usr/local/jenkins-service
					echo db49da756c17a908933afd5e24bd4712500c8ac5f46e9c5b9418cd4787d9b31c > secret-file
					curl -sO http://172.28.160.2:8080/jnlpJars/agent.jar
					java -jar agent.jar -url http://172.28.160.2:8080/ -secret @secret-file -name "jenkins-node" -webSocket -workDir "/var/lib/jenkins"
					rm secret-file
					exit 0
					```
	    				- `sudo chmod +x /usr/local/jenkins-service/start-agent.sh`
     				- create and enable systemd service
					- `sudo nano /etc/systemd/system/jenkins-agent.service`
					```
					[Unit]
					Description=Jenkins Agent
					
					[Service]
					User=jenkins
					Group=jenkins
					WorkingDirectory=/var/lib/jenkins
					ExecStart=/bin/bash /usr/local/jenkins-service/start-agent.sh
					Restart=always
					
					[Install]
					WantedBy=multi-user.target
					```
					- `sudo systemctl daemon-reload`
					- `sudo systemctl enable jenkins-agent.service`
					- `sudo service jenkins-agent start`
	- setup pipeline
	```
	composer update
	composer test
	composer stan
	composer cs
	composer cs-fix
	export PATH/usr/bin:$PATH
	export DOCKER_HOST=unix:///run/user/1001/docker.sock
	...
	```
- docker setup
	- `sudo apt install ca-certificates curl`
	- `sudo install -m 0755 -d /etc/apt/keyrings`
	- `sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`
	- `sudo chmod a+r /etc/apt/keyrings/docker.asc`
	```
   	echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	```
	- `sudo apt update`
	- `sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
	- docker rootless mode
		- `sudo apt install uidmap dbus-user-session systemd-container docker-ce-rootless-extras`
		- `sudo systemctl disable --now docker.service docker.socket`
		- execute setup script bundled with DEB package as `ubuntu` user
			- `sudo machinectl shell TheUser@`
			- `/usr/bin/dockerd-rootless-setuptool.sh install`
		- verify local user service deployment status
			- `systemctl --user status docker`
		- enable deamon autolaunch on startup
			- `systemctl --user enable docker`
			- `sudo loginctl enable-linger ubuntu`
	- setup custom docker registry container
		- `docker run --restart=always -d -p 172.28.160.2:5000:5000 --name registry registry:latest`
- php (without `libapache2` dependency)
	- `sudo apt install php php8.3-fpm php-pgsql php-xml php-mbstring`
- composer
	- `sudo apt install composer`
			
### roadrunner
- docker setup
	- `sudo apt install ca-certificates curl`
	- `sudo install -m 0755 -d /etc/apt/keyrings`
	- `sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`
	- `sudo chmod a+r /etc/apt/keyrings/docker.asc`
	```
   	echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	```
	- `sudo apt update`
	- `sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
- docker rootless mode
	- `sudo apt install uidmap dbus-user-session systemd-container docker-ce-rootless-extras`
	- `sudo systemctl disable --now docker.service docker.socket`
	- execute setup script bundled with DEB package as `ubuntu` user
		- `sudo machinectl shell TheUser@`
		- `/usr/bin/dockerd-rootless-setuptool.sh install`
	- verify local user service deployment status
		- `systemctl --user status docker`
	- enable deamon autolaunch on startup
		- `systemctl --user enable docker`
		- `sudo loginctl enable-linger ubuntu`
	
### postgres
- via manually configured APT repository
	- `sudo apt install curl ca-certificates`
	- `sudo install -d /usr/share/postgresql-common/pgdg`
	- `sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc`
	- `sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`
	- `sudo apt update`
	- `sudo apt install postgresql-15`
	- modify `/etc/postgresql/15/main/postgresql.conf` file
		- `listen_address = "172.28.160.4"`
	- modify `/etc/postgresql/15/main/pg_hba.conf`
		- `host all all 172.28.160.4/28 scram-sha-256`
		- `host replication all 172.28.160.4/28 scram-sha-256`
		- disable other `host`/`local` entries which we're not going to use (**except for unix domain socket**)
		- `sudo service postgresql restart`
	- connect to default template DB and modify postgres password
		- `sudo -u postgres psql template1`
		- `ALTER USER postgres with encrypted password '<password>';`
	- verify connectivity
		- `sudo apt install postgresql-client`
		- `psql --host 172.28.160.4 --username`
	- create user/db app
		- `create database app;`
		- `create user app with encrypted password '<password>';`
		- `grant all privileges on database app to app;`
			
### redis			
- via manually configured APT repository
	- `curl https://packages.redis.io/gpg | sudo apt-key add -`
	- `echo "deb https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list`
	- `sudo apt update`
	- in order to avoid unmet dependencies error, specify version in all related packages
		- `sudo apt install redis=6:6.2.17-1rl1~noble1 redis-server=6:6.2.17-1rl1~noble1 redis-tools=6:6.2.17-1rl1~noble1`
	- `sudo systemctl enable redis-server`
	- modify `/etc/redis/redis.conf` file
		- `bind 172.28.160.5`
		- `requirepass <password>`
		- `appendonly yes`
		- `appendfile "appendonly.aof"`
	- `sudo service redis-server restart`
	- verify connectivity
		- `redis-cli -h 172.28.160.5`
		- `auth <password>`
	
## Troubleshooting
- Jenkins builds failing
	- `Script cache:clear returned error code 1`
		```
		!!  
		!!  In FileLoader.php line 182:
		!!                                                                                 
		!!    There is no extension able to load the configuration for "baldinof_road_run  
		!!    ner" (in "/var/lib/jenkins/workspace/rcs/config/packages/baldinof_road_runn  
		!!    er.yaml"). Looked for namespace "baldinof_road_runner", found ""framework",  
		!!     "doctrine", "doctrine_migrations", "maker"" in /var/lib/jenkins/workspace/  
		!!    rcs/config/packages/baldinof_road_runner.yaml (which is being imported from  
		!!     "/var/lib/jenkins/workspace/rcs/src/Kernel.php").                           
		!!                                                                                 
		!!  
		!!  In YamlFileLoader.php line 808:
		!!                                                                                 
		!!    There is no extension able to load the configuration for "baldinof_road_run  
		!!    ner" (in "/var/lib/jenkins/workspace/rcs/config/packages/baldinof_road_runn  
		!!    er.yaml"). Looked for namespace "baldinof_road_runner", found ""framework",  
		!!     "doctrine", "doctrine_migrations", "maker"".                                
		!!                                                                                 
		!!  
		!!  
		```
		- add line to `config/bundles.php`
			- `Baldinof\RoadRunnerBundle\BaldinofRoadRunnerBundle::class => ['all' => true]`
		- instead of `composer install` run `composer update` command which will fix missing dependency/dependency mismatch errors caused by `composer.lock` or `composer.json` files
	- `[critical] Error thrown while running command "doctrine:migrations:migrate". Message: "An exception occurred while executing a query: SQLSTATE[42501]: Insufficient privilege: 7 ERROR:  permission denied for schema public`
		- make sure to set `app` user as owner of both `app` database and schema `public`. after that migration wil run normally

	- missing `Dockerfile`/`.rr.yaml` files after build
		- no solution yet
