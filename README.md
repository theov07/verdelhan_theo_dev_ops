# DevOps Verdelhan Théo


# TP1 - Docker


## Folder structure
- sql-scripts/ : Contains configuration files and SQL scripts for the database.
- simple-api-student-main/ : Contains the source code and Dockerfile for the backend API.
- http-server/: Contains the HTML file and Dockerfile for the HTTP server.
- docker-compose.yml: Configuration file for Docker Compose.


## Objectives
1.	Configure and launch a PostgreSQL database in a Docker container.
2.	Develop a Java backend API and deploy it in a Docker container.
3.	Create a simple HTTP server and configure it as a reverse proxy.
4.	Use Docker Compose to manage and orchestrate containers.




## 1. Database



### 1. Build the PostgreSQL Image

```bash
docker build -t mypostgres:v1.0 ./database
```


### 2. Create the Docker Network

```bash
docker network create app-network
```


### Question 

Why should we run the container with a flag -e to give the environment variables?

reponse : We should run the container with the -e flag to provide environment variables because it allows for secure and flexible configuration management without hardcoding sensitive information in the Dockerfile.


### 3. Run the PostgreSQL Container on the Network

```bash
docker run --name my-db -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -p 5432:5432 --network app-network -v /my/own/datadir:/var/lib/postgresql/data -d mypostgres:v1.0
```

### 4. Execution script verification 

```bash
docker exec -it my-db psql -U usr -d db
```


### 6. Run Adminer on the Network

```bash
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

To connect to the PostgreSQL database, go to https://localhost:8090 in your browser and use the following credentials.

- Server: my-db
- User: usr
- Password: pwd
- Database: db



### Question 

Why do we need a volume to be attached to our postgres container?

reponse : We need a volume to be attached to our PostgreSQL container to ensure data persistence, allowing the database to retain information even if the container is stopped or deleted.


Why do we need a multistage build? And explain each step of this Dockerfile.

reponse : A multistage build is needed to optimize the Docker image by separating the build environment from the runtime environment, resulting in a smaller, more secure, and efficient final image.





## 2. Backend API


### Target java 

```bash
javac Main.java
```


### DockerFile API 


```bash
FROM maven:3.8.6-amazoncorretto-17 AS build
WORKDIR /app
COPY pom.xml /app
RUN mvn dependency:go-offline
COPY src /app/src
RUN mvn package -DskipTests


FROM amazoncorretto:17
WORKDIR /app
COPY --from=build /app/target/*.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```


### Build springboot image

```bash
docker build -t springboot-app .
```


### Dockerfile backend_api 


```bash
FROM openjdk:23-slim-bookworm

COPY ./Main.class /usr/src/myapp/Main.class

WORKDIR /usr/src/myapp

CMD ["java", "Main"]
```


### Run the container

```bash
docker run --name simple_api_student -p 8080:8080 --network app-network springboot-app-mai
```



### Question 

Why do we need a multistage build? And explain each step of this Dockerfile.

reponse : A multistage build is needed to optimize the Docker image by separating the build environment from the runtime environment, resulting in a smaller, more secure, and efficient final image.


### Check point

We go to https://localhost:8080/departments/IRC/students and we see that : 


```json
[
  {
    "id": 1,
    "firstname": "Eli",
    "lastname": "Copter",
    "department": {
      "id": 1,
      "name": "IRC"
    }
  }
]
```



## 3. Http server

### Index HTML file 


```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome</title>
</head>
<body>
  <h1>Hello, welcome to my simple HTTP server!</h1>
</body>
</html>
```



### Pull httpd image 

```bash
docker pull httpd
```


### Build httpd image

```bash
docker build -t my-http-server .
```


### Run the container

```bash
docker run -d -p 80:80 --network app-network --name my-web-server my-http-server
```

### Copy the configuration into httpd.conf file in the container

```bash
docker cp httpd.conf my-web-server:/usr/local/apache2/conf/httpd.conf
```



### Question 

Why do we need a reverse proxy?

reponse : We need a reverse proxy to route client requests to the appropriate backend service, enhance security, enable load balancing, and provide a single entry point for the application.



## 4. Docker Compose file 


```yaml
version: '3.7'
 
services:
    backend:
        build: ./simple-api-student-main
        container_name: "simple_api_student"

        environment:
            DB_host: database
            DB_port: 5432
            DB_name: db
            DB_user: usr
            DB_mdp: pwd

        ports:
          - "8080:8080"
        networks:
          - app-network
        depends_on:
          - database
 
 
    database:
        
        build: ./sql-scripts
        container_name: "database"

        environment:
          POSTGRES_DB: db
          POSTGRES_USER: usr
          POSTGRES_PASSWORD: pwd
        
        ports:
          - "5432:5432"
          
        networks:
          - app-network
 

    httpd:
        
        build: ./http-server
        ports:
          - "80:80"
        networks:
          - app-network
        depends_on:
         - backend
         - database
 
networks:
    app-network:
```



### Run Docker compose file

```bash
docker compose up
```

### Question 

Why is docker-compose so important?

reponse : Docker Compose is important because it simplifies the management of multi-container Docker applications by allowing you to define and orchestrate all your services in a single configuration file, facilitating development, testing, and deployment processes.



## 5. Publish

### Create tag for DockerHub 


```bash
docker tag mypostgres:v1.0 theov07/my-database:lastest
```

```bash
docker tag springboot-app-main:latest theov07/api_backend:v1.0 
```

```bash
docker tag my-http-server:latest theov07/my-http-server:v1.0
```



### Push on DockerHub 

```bash
docker push theov07/my-database:lastest
```

```bash
docker push theov07/api_backend:v1.0
```

```bash
docker push theov07/my-http-server:v1.0
```



### Question 

Why do we put our images into an online repo?

reponse : We put our images into an online repo to enable easy sharing, versioning, and deployment across different environments and team members, ensuring consistent and reproducible builds.






# TP2 - Github Actions



### Build and test your Application


```bash
mvn clean verify --file simple-api-student-main/pom.xml
```


```yaml
name: CI devops 2024

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test-backend:
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=devopstheo2_devops -Dsonar.organization=devopstheo2 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file simple-api-student-main/pom.xml

  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api-student-main
          tags: ${{secrets.DOCKERHUB_USERNAME}}/api_backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./sql-scripts
          tags: ${{secrets.DOCKERHUB_USERNAME}}/my-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{secrets.DOCKERHUB_USERNAME}}/my-http-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}

```


### Explanation of the main file in the workflow 


1.	This GitHub Actions workflow runs on pushes or pull requests to the main and develop branches.
2.	It includes two jobs: test-backend for testing the backend with Maven and SonarCloud, and build-and-push-docker-image for building and pushing Docker images.
3.	The test-backend job checks out the code, sets up JDK 17, and runs Maven to verify the code and perform a SonarCloud analysis.
4.	The build-and-push-docker-image job depends on the test-backend job, and it checks out the code, logs into DockerHub, and builds and pushes Docker images for the backend, database, and HTTP server if the push is to the main branch.




### Adding the SonarCloud Token to Secrets

Go to your repository's settings.
Navigate to "Secrets" or "Variables" (depending on your CI platform).
Add a new secret named SONAR_TOKEN and set its value to your SonarCloud token.




# TP3 - Ansible


### Install Ansible

```bash
brew install ansible
```



### Check Ansible version

```bash
ansible --version
```



### Set Permissions for SSH Key

```bash
chmod 400 id_rsa
```


### Connect to the host

```bash
ssh -i id_rsa centos@theo.verdelhan.takima.cloud
```


### Install vim in the host

```bash
sudo yum install vim -y
```



### Run setup file with ssh key

```bash
ansible -i ansible/inventory/setup.yml all -m ping --private-key=id_rsa -u centos
```

after that we receive success with Pong response



### Installing Apache via Ansible

```bash
ansible -i ansible/inventory/setup.yml all -m yum -a "name=httpd state=present" --private-key=id_rsa -u centos --become
```




### Updating the Home Page with Ansible

```bash
ansible prod -i ansible/inventory/setup.yml -m shell -a 'echo "<html><h1>Hello World</h1></html>" >> /var/www/html/index.html' --become
```


### Start Apache via Ansible

```bash
ansible all -i ansible/inventory/setup.yml  -m service -a "name=httpd state=started" --become 
```


we go on https://theo.verdelhan.takima.cloud and we see "HELLO WORLD"





### Get distribution details via Ansible

```bash
ansible all -i ansible/inventory/setup.yml -m setup -a "filter=ansible_distribution*”
```




### Remove Apache via Ansible

```bash
ansible all -i ansible/inventory/setup.yml -m yum -a "name=httpd state=absent" --become
```



### Start Ansible playbook

```bash
ansible-playbook -i ansible/inventory/setup.yml ansible/playbook.yml
```



### Final Ansible playbook

```yaml
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:

  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker



# App Builder using the Docker role
# - hosts: all
#   become: true
#   roles: 
#     - docker


- hosts: all
  become: true
  vars:
    ansible_python_interpreter: /usr/bin/python3

  roles:
    - install_docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy

```


### Playbook explaination 


1.	The playbook runs on all hosts and escalates privileges using become, but it does not gather facts.
2.	It installs necessary packages for Docker, including device-mapper-persistent-data and lvm2, and adds the Docker repository.
3.	Docker and Python 3 are installed, and the Docker Python module is installed using pip3.
4.	The Docker service is ensured to be running.
5.	The playbook then uses various roles to install Docker, create a network, and launch the database, application, and proxy services.




### Creating a Role for Docker Installation

```bash 
ansible-galaxy init roles/docker
```



### install_docker role, main file

```yaml
---
- name: Install required packages
  yum:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
    state: present

- name: Add Docker repository
  command: yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Start Docker service
  service:
    name: docker
    state: started
    enabled: yes
```




### create_network role, main file

```yaml
---
- name: Create Docker network
  docker_network:
    name: app-network
```




### lauch_database role, main file

```yaml
---
- name: Launch database container
  docker_container:
    name: database
    image: theov07/my-database:latest
    ports:
      - "5432:5432"
    networks:
      - name: app-network
```






### lauch_app role, main file

```yaml
---
- name: Launch backend container
  docker_container:
    name: simple_api_student
    image: theov07/api_backend:latest
    ports:
      - "8080:8080"
    networks:
      - name: app-network
```






### lauch_proxy role, main file

```yaml
---
- name: Launch httpd container
  docker_container:
    name: httpd
    image: theov07/my-http-server:latest
    ports:
      - "80:80"
    networks:
      - name: app-network
```


### Deploy all the application 

```bash
ansible-playbook -i ansible/inventory/setup.yml ansible/playbook.yml
```




### Workflow deploy.yml for continous deployment


```yaml
name: Deploy Application

on:
  workflow_run:
    workflows: ["CI devops 2024"]
    types:
      - completed

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Set up SSH
      uses: webfactory/ssh-agent@v0.5.3
      with:
        ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

    - name: Install Ansible
      run: |
        sudo apt-get update
        sudo apt-get install -y ansible

    - name: Disable host key checking
      run: echo "ANSIBLE_HOST_KEY_CHECKING=False" >> $GITHUB_ENV

    - name: Run deployment playbook
      run: |
        ansible-playbook -i ansible/inventory/setup.yml ansible/playbook.yml
      env:
        ANSIBLE_HOST_KEY_CHECKING: False

```



### Workflow deploy explaination 

1.	This workflow is triggered by the completion of the “CI devops 2024” workflow and runs only if it is successful.
2.	It defines a job named deploy that runs on the latest Ubuntu runner.
3.	The job checks out the repository and sets up SSH using a private key stored in GitHub Secrets.
4.	Ansible is installed on the runner, and host key checking is disabled.
5.	Finally, the workflow runs an Ansible playbook to deploy the application.