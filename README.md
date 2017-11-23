# Quick guide on how to install Drone Server 0.4 on AWS and how to use a local Drone Client to access it

## Drone Build Server

1.- Create an Ubuntu 16.04 instance in EC2, with a Security Group that allows SSH, HTTP and HTTPS traffic. 

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

3.- Install Nginx

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
	
4.- Secure Nginx with Let's Encrypt

	Create a fully registered domain name (free on freenom.com) with an A record for yourdomain.com pointing to the public IP address of your EC2 server, and another A record for www.yourdomain.com pointing to the same IP
	
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

5.- Register Drone app in GitHub

	Click on User - Settings 
	Go to Developer setting in the lower left 
	OAuth Applications - Register a new application
	Name, http://www.yourdomain.com, Description, http://www.yourdomain.com/authorization
	Register application, and get your ‘Client ID’ (DRONE_GITHUB_CLIENT) and ‘Client Secret’ (DRONE_GITHUB_SECRET)

6.- Install Drone Server 0.4

	sudo mkdir /etc/drone
	sudo vi /etc/drone/dronerc

		REMOTE_DRIVER=github
		REMOTE_CONFIG=https://github.com?client_id=DRONE_GITHUB_CLIENT&client_secret=DRONE_GITHUB_SECRET
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

	sudo vi /etc/systemd/system/drone.service
	
		[Unit]
		Description=Drone server
		After=docker.service nginx.service

		[Service]
		Restart=always
		ExecStart=/usr/local/bin/docker run --volume /var/lib/drone:/var/lib/drone --volume /var/run/docker.sock:/var/run/docker.sock --env-file /etc/drone/dronerc --restart=always --publish=8000:8000 --detach=true --name=drone drone/drone:0.4
		ExecStop=/usr/local/bin/docker rm -f drone

		[Install]
		WantedBy=multi-user.target

7.- Configure Nginx to proxy requests to Drone

	grep -R server_name /etc/nginx/sites-enabled
		server_name yourdomain.com www.yourdomain.com;
	sudo vi /etc/nginx/sites-enabled/default
		add this OUT of the server block:
			upstream drone {
			    server 127.0.0.1:8000;
			}

			map $http_upgrade $connection_upgrade {
			    default upgrade;
			    ''      close;
			}

			server {
			    . . .
		replace the content of 'location' under server listening for 443 ssh
			location / {
				# try_files $uri $uri/ =404;
				proxy_pass http://drone;

				include proxy_params;
				proxy_set_header Upgrade $http_upgrade;
				proxy_set_header Connection $connection_upgrade;

				proxy_redirect off;
				proxy_http_version 1.1;
				proxy_buffering off;
				chunked_transfer_encoding off;
				proxy_read_timeout 86400;
			}

8.- Test and restart Nginx and Drone

	sudo nginx -t
	sudo systemctl restart nginx
	sudo systemctl start drone
	sudo systemctl status drone


## Drone Client


1.- Run CICD client demo container (it includes nano, git, docker, drone cli tools). If you need the container to RUN containers: docker run -it --name cicdlab -v /var/run/docker.sock:/var/run/docker.sock hpreston/devbox:cicdlab

	docker run -it --name cicdlab hpreston/devbox:cicdlab
	
2.- Fork imapex-training/cicd_demoapp github repo and clone your copy to your local env, and create docker repo with the same name

3.- Activate the repo in the Drone server

    make sure the lab admin has enabled your github account in the lab server
    go to yourdomain.com, login / authorize drone to access your repo, activate repo
    
4.- Build secrets file and push it to Git

    cd ~/coding/cicd_demoapp
    export DRONE_SERVER=http://yourdomain.com
    export DRONE_TOKEN=<your_token>
        (token can be found by going to yourdomain.com, click on the user, go to profile, click on show token and copy the value)
    drone repo ls
    cp drone_secrets_sample.yml drone_secrets.yml
    vi drone_secrets.yml
        (complete it with the required info but DO NOT ADD / COMMIT IT TO YOUR REPO)
	
    Every time you change the .drone.yml file:
        drone secure --repo <username>/cicd_demoapp --in drone_secrets.yml
            (this creates an encrypted .drone.sec file)
        git add .drone.sec
        git add .drone.yml
        git commit -m "<your_comment>"
        git push
