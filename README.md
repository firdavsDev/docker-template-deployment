# This is a documentation of how to deploy a project on AWS/Ubuntu server

## Setup


- update the OS:

```
- sudo apt-get update
```

- Install git *If it does not already exist*

```
- git --version
- sudo apt install git
```

- Install docker:
  - set up and install prerequisites
  - update the system
  - install docker
  - add docker to sudouser

```
Installing prerequisites:
- sudo apt-get remove docker docker-engine docker.io containerd runc
- sudo apt-get update
- sudo apt-get install ca-certificates curl gnupg lsb-release
- sudo mkdir -p /etc/apt/keyrings
- curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
- echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

Updating the system:
- sudo apt-get update

Installing docker:
- sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

Adding docker to sudouser:
- sudo usermod -aG docker $USER
```

- Installing docker-compose

```
- sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
- sudo chmod +x /usr/local/bin/docker-compose
- docker-compose --version
```

- Note
```
Log out and log back in so that docker is added to sudo user
```

- Now, it is time to clone the project
  - in order to clone the project, we need to generate a key and add it to our github SSH/GPG key section as new key
  - in order to add the key follow the instruction below


- Set up SSH key based authentication

```
assuming that we are in our virtual server:
- sudo apt update && sudo apt install -y openssh-client git
- mkdir -p -m 700 ~/.ssh/github
- ssh-keygen -t rsa -b 4096 -C 'your@email.com' -f ~/.ssh/github/id_rsa -q -N ''
- cat ~/.ssh/github/id_rsa.pub

Copy the printed key and navigate to:
- From the drop-down menu in upper right corner select Settings
- Then from the menu at the left side select SSH and GPG keys
- Click on the New SSH Key button
- Type some meaningful for a Title and paste the content of ~/.ssh/github/id_rsa.pub in the field Key
- Then click on the Add SSH Key button

Back into our virtual server:
- touch ~/.ssh/config
- chmod 600 ~/.ssh/config
- vim ~/.ssh/config

----------------copy below the two line and paste int into vim---------------------
Host github.com
    IdentityFile ~/.ssh/github/id_rsa
----------------copy above the two line and paste int into vim---------------------

In order to save and leave the vim:
- press Esc (escape button) couple of times
- press "shift and :" key
- type "wq", it save the current state of vim

Check if everything is fine:
- ssh -T git@github.com
  it should prine a message like this:
  Hi <your_username> You've successfully authenticated, but GitHub does not provide shell access
```

- Note!
  - Up to this point, configuration should have been done successfully
  - Now it is time to deploy our project


- Deploying the project
```
- cd <your project's root folder> where docker-compose files lie
- docker-compose -f production.yml up --build -d 
```


- Getting a dump from database
```
docker exec -t <database-container> pg_dump -c -U <database-user> -d <database-name> > dump_`date +%d-%m-%Y"_"%H_%M_%S`.sql 
```

- Copying a file from AWS EC2 to local machone
```
scp -i <your-pem-key.pem> root@IP:path/to/your/dump/dump.sql /local/machine/directory
```

- Restoring a DB
```
cat <dump.sql> | docker exec -i <database-container> psql -U <database-user> -d <database-name>
```
