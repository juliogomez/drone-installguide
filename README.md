# Quick reference guide on how to install Drone

## Drone Build Server

1.- Create EC2 instance, with SG opening SSH & TCP. Its DNS name (EC2_URL) will be used as DRONE_HOST later in step 5.

2.- Install docker

	sudo yum update -y
	sudo yum install -y docker
	sudo service docker start
	sudo usermod -a -G docker ec2-user
	log out and back in

3.- Install docker-compose

	sudo curl -L https://github.com/docker/compose/releases/download/1.17.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
		(check latest from https://github.com/docker/compose/releases)
	sudo chmod +x /usr/local/bin/docker-compose

4.- Configure GitHub

	Click on User - Settings 
	Go to Developer setting in the lower left 
	OAuth Applications - Register a new application
	Name, http://EC2_URL, Description, http://EC2_URL/authorization
	Register application, and get your ‘Client ID’ (DRONE_GITHUB_CLIENT) and ‘Client Secret’ (DRONE_GITHUB_SECRET)

5.- Install Drone

	docker pull drone/drone:0.7
	sudo mkdir /etc/drone
	sudo vi /etc/drone/docker-compose.yml

    version: '2'

    services:
      drone-server:
        image: drone/drone:0.7
        ports:
          - 80:8000
        volumes:
          - /var/lib/drone:/var/lib/drone/
        restart: always
        environment:
          - DRONE_OPEN=true
          - DRONE_HOST=${DRONE_HOST}
          - DRONE_GITHUB=true
          - DRONE_GITHUB_CLIENT=${DRONE_GITHUB_CLIENT}
          - DRONE_GITHUB_SECRET=${DRONE_GITHUB_SECRET}
          - DRONE_SECRET=${DRONE_SECRET}
    
      drone-agent:
        image: drone/drone:0.7
        command: agent
        restart: always
        depends_on:
          - drone-server
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        environment:
          - DRONE_SERVER=ws://drone-server:8000/ws/broker
          - DRONE_SECRET=${DRONE_SECRET}


## Drone Client


1.- Install Drone CLI (Mac)

     curl http://downloads.drone.io/drone-cli/drone_darwin_amd64.tar.gz | tar zx
     sudo cp drone /usr/local/bin

2.- Create github and docker repos with the same name

3.- Activate the repo in Drone

    make sure the lab admin has enabled your github account in the lab server
    go to drone server url, login / authorize drone, activate repo
    
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