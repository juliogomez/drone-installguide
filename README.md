# Quick reference guide to install Drone Server and Client

## Drone Build Server

1.- Create an Ubuntu 16.04 instance in EC2, with SG opening SSH & TCP. Its DNS name (EC2_URL) will be used as DRONE_HOST later.

2.- Install docker

	Option 1 (tested)
		curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
		sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
		sudo apt-get update
		apt-cache policy docker-ce
		sudo apt-get install -y docker-ce
		sudo systemctl status docker

	Option 2 (untested)
		sudo yum update -y
		sudo yum install -y docker
		sudo service docker start

	sudo usermod -a -G docker ec2-user
	(log out and back in)

3.- Install docker-compose (optional)

	sudo curl -L https://github.com/docker/compose/releases/download/1.17.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
		(check what's the latest version from https://github.com/docker/compose/releases)
	sudo chmod +x /usr/local/bin/docker-compose

4.- Install Nginx

	sudo apt-get update
	sudo apt-get install nginx
	sudo ufw status (check it is inactive)
	systemctl status nginx (check nginx status)
	
	Option 1:
		ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//' (check your IP and try access from browser)
	Option 2:
		sudo apt-get install curl
		curl -4 icanhazip.com (check your IP and try access from browser)

	Manage Nginx with:
		sudo systemctl stop/start/restart/reload nginx
	Default html content served from: /var/www/html
	Nginx configuration stored here: /etc/nginx
	
5.- Secure Nginx with Let's Encrypt

	Create a fully registered domain name (free on freenom.com) with an A record for yourdomain.com pointing to the public IP address of your server, and another A record for www.yourdomain.com pointing to the same IP
	
	sudo add-apt-repository ppa:certbot/certbot
	sudo apt-get update
	sudo apt-get install python-certbot-nginx
	sudo nano /etc/nginx/sites-available/default
		find server_name and replace _ with yourdomain.com www.yourdomain.com;
	sudo nginx -t
	sudo systemctl reload nginx
	
	sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
	
	Try reloading your website using https://
	
	sudo certbot renew --dry-run

4.- Configure GitHub

	Click on User - Settings 
	Go to Developer setting in the lower left 
	OAuth Applications - Register a new application
	Name, http://EC2_URL, Description, http://EC2_URL/authorization
	Register application, and get your ‘Client ID’ (DRONE_GITHUB_CLIENT) and ‘Client Secret’ (DRONE_GITHUB_SECRET)

5.- Install Drone Server

	sudo mkdir /etc/drone
	sudo vi /etc/drone/dronerc

		REMOTE_DRIVER=github
		REMOTE_CONFIG=https://github.com?client_id=${client_id}&client_secret=${client_secret}
		DATABASE_DRIVER=sqlite3
		DATABASE_CONFIG=/var/lib/drone/drone.sqlite
	
	sudo docker run \
	--volume /var/lib/drone:/var/lib/drone \
	--volume /var/run/docker.sock:/var/run/docker.sock \
	--env-file /etc/drone/dronerc \
	--restart=always \
	--publish=8000:8000 \
	--detach=true \
	--name=drone \
	drone/drone:0.4


## Drone Client


1.- Install Drone CLI (Mac)

     curl http://downloads.drone.io/drone-cli/drone_darwin_amd64.tar.gz | tar zx
     sudo cp drone /usr/local/bin

2.- Create github and docker repos with the same name

3.- Activate the repo in the Drone server

    make sure the lab admin has enabled your github account in the lab server
    go to EC2_URL, login / authorize drone, activate repo
    
4.- Build secrets file

    cd to your local repo
    export DRONE_SERVER=http://EC2_URL
    export DRONE_TOKEN=<your_token>
        (token can be found by going to EC2_URL, click on the user, go to profile, click on show token and copy the value)
    drone repo ls
    vi drone_secrets.yml
    
    environment:
    SPARK_TOKEN: <FROM YOUR DEVELOPER.CISCOSPARK.COM ACCOUNT>
    SPARK_ROOM: <EITHER ONE OF YOUR OWN ROOMS, OR A ROOMID PROVIDED BY THE LAB ADMIN>
    DOCKER_USERNAME: <YOUR HUB.DOCKER.COM USERNAME>
    DOCKER_PASSWORD: <YOUR HUB.DOCKER.COM PASSWORD>
    DOCKER_EMAIL: <YOUR HUB.DOCKER.COM EMAIL ADDRESS>
    MANTL_USERNAME: <MANTL USER PROVIDED BY LAB ADMIN>
    MANTL_PASSWORD: <MANTL PASSWORD PROVIDED BY LAB ADMIN>
    MANTL_CONTROL: <MANTL SERVER ADDRESS PROVIDED BY LAB ADMIN>
    
    (DO NOT ADD / COMMIT IT TO YOUR REPO)
    
    drone secure --repo <username>/<repo> --in drone_secrets.yml
