This application is for the gathering, storage, and display of Reddit data. 
I love large structures - behemoth applications which include a lot of moving parts. 

This is one of them.

## A little bit of history

I worked at a company called Digital First Media. I was hired on as a data optimization engineer. 
The job was primary working on their optimization code for online marketing campaigns. As the guy in-between, I worked
with qualified data engineers on one side of me and creative web developers on the other side.
 
[Duffy](https://github.com/duffn) was definitely one of the talented ones and taught me quite a bit. While I was with 
the company, one of my complaints was related to how much we were spending for tools we could easily make in house. Of 
of those choices, was whether to buy RSConnect or not. I found a way to build highly scalable R APIs using 
docker-compose and NGINX. Duffy was the guy who knew what was needed for a solution, so he gave me quite a bit of 
guidance in understanding good infrastructure. 

So that's where I learned about building really cool APIs so that I could share my output with non R users. Well, there 
are people who my APIs, and Duffy was doing cool stuff in Data Engineering and was getting the data to me so I could
build the API. I gravitated a bit out of the math into the tools Data Engineers used, and became interested in Python, 
SQL, Airflow etc. These guys spin that stuff daily, and so it's not impossible to learn! I started creating data 
pipelines, which grew - and became difficult to maintain. I wanted to learn best practices in data engineering - because 
when things break, it's devastating and a time sink and kept me up nights.  

AIRFLOW, is one of the tools for this job. It makes your scheduled jobs smooth like butter, and is highly transparent 
with the health of your network, and allows for push button runs of your code. This was far superior to cron.

Well, I learned data engineering stuff and wanted to learn React/Javascript. This is the most recent venture, and I'm 
still learning. 
                                                                                                                          
                                                                                                                          
## About This Project

The main components are 

1. An Airflow instance running scripts for data gathering.
2. A Postgres database to store the data, with scheduled backups to AWS S3.
3. An R Package I wrote for talking to AWS called `biggr` (using a Python backend - its an R wrapper for `boto3` using Reticulate)
4. An R Package I wrote for talking to Reddit called `redditor`  (using a Python backend - its an R wrapper for `praw` using Reticulate)  
5. An R API that converts the data generated by this pipeline to a front end application for display
6. A React Application which takes the data in the R API and displays it on the web.

## You will Need
1. Reddit API authentication
2. AWS IAM Creds
3. Motivation to learn Docker, PostgreSQL, MongoDB, Airflow, R Packages using Reticulate, Load Balancing, 
 and web design.
4. Patience while Docker builds
5. An interest in programming
6. A pulse
7. Oxygen
8. Oreos

## Cleaning up for Installation
If you are developing some part of R code, there are two scripts which you can adapt to your particular filepath


## Copy Pastin'
There are three dockerfiles that are needed: `DockerfileApi`, `DockerfileRpy`, and `DockerfileUi`

`DockerfileApi` is associated with the container needed to run an R [Plumber](https://www.rplumber.io/) API. 
In the container I take from [trestletech](https://hub.docker.com/r/trestletech/plumber/), I add on some additional 
Linux binaries and R packages. There are two R packages in this project. One is called [biggr] and the other is called 
[redditor], which are located in `./bigger` and `./redditor-api` respectively. To build the container, run the 
following:

```
docker build -t redditorapi --file ./DockerfileApi .
```

`DockerfileRpy` is a container running both R and Python, This is taken from the `python:3.7.6` container. I install R 
on top of it, so I can run scheduled jobs. This container runs Airflow, which is set up in `airflower`. Original name, 
right? 

```
docker build -t rpy --file ./DockerfileRpy .
```

This container contains node, npm, and everything else needed to run the site. The site is a React application using 
Material UI. The project is located at `redditor-ui`

```
docker build -t redditorui --file ./DockerfileUi .
```

All of the above gets our containers ready for use. But there's a lot to unpack in this docker-compose file.

### LOTS OF SERVICES

```
services:
```

The first is the web server. This hosts NGINX, which is being used as a load balancer and reverse proxy. A load 
balancer takes incoming traffic, and routes that traffic to multiple different locations, Say you clone an API, as I
do in the project. You can then do the same or different tasks simultaneously. If there is one API, then each process 
has to 'wait in line' until the one ahead of it is done. 

So, you arrive to `localhost/api` and it routes you to either port 8000 or 8001, where the R apis live.
If you arrive to `localhost/airflowr` then you'll see airflow running. How this is done is in `nginx.conf` and the web
service + `nginx.conf` file can be used alone for services requiring the same types of configuration.

```
  web:
    image: nginx
    volumes:
     - ./nginx.conf:/etc/nginx/conf.d/mysite.template
    ports:
     - "80:80"
    environment:
     - NGINX_HOST=host.docker.internal
     - NGINX_PORT=80
    command: /bin/bash -c "envsubst < /etc/nginx/conf.d/mysite.template > /etc/nginx/nginx.conf && exec nginx -g 'daemon off;'"
```


This runs the Airflow UI, which is located at `localhost:8080`

```
  webserver:
    image: rpy
    restart: always
    depends_on:
      - initdb
    env_file: .env
    environment:
      AIRFLOW_HOME: /root/airflow
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@host.docker.internal:5439/airflow
    volumes:
      - ./airflower/dags:/root/airflow/dags
      - ./airflower/plugins:/root/airflow/plugins
      - airflow-worker-logs:/root/airflow/logs
    ports:
      - "8080:8080"
    command: airflow webserver
```

This hosts the `MongoDB` at the password specified in `./init-mongo.js/init-mongo.js` and looks something like so


```
db.createUser(
    {
        user: "fdrennan",
        pwd: "password",
        roles: [
            {
                role: "admin",
                db: "admin"
            }
        ]
    }
);
```

and coordinate to your `.env` file in the root directory which is driven in the following service.

```
MONGO_INITDB_DATABASE=admin
MONGO_INITDB_ROOT_USERNAME=fdrennan
MONGO_INITDB_ROOT_PASSWORD=password
```

```
  mongo_db:
    image: 'mongo'
    container_name: 'ndexr_mongo'
    restart: always
    ports:
      - '27017:27017'
    expose:
      - '27017'
    volumes:
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
      - mongodbdata:/data/db
    environment:
      - MONGO_INITDB_DATABASE=${MONGO_INITDB_DATABASE}
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_INITDB_ROOT_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_INITDB_ROOT_PASSWORD}
```


Next we build our PostgreSQL database which has the following environment variables and need to be in the .env file.

```
AIRFLOW_USER=airflow
AIRFLOW_PASSWORD=airflow
AIRFLOW_DB=airflow
```

```
  postgres:
    image: postgres:9.6
    restart: always
    environment:
      - POSTGRES_USER=${AIRFLOW_USER}
      - POSTGRES_PASSWORD=${AIRFLOW_PASSWORD}
      - POSTGRES_DB=${AIRFLOW_DB}
    ports:
      - 5439:5432
    volumes:
      - postgres:/var/lib/postgresql/data
      - ./data/postgres.bak:/postgres.bak
```

I dont know WTF this is. I copy paste stuff too, ya know. Need it to work, must work, since it works. :)
 
```
  initdb:
    image: rpy
    restart: always
    depends_on:
      - postgres
    env_file: .env
    environment:
      AIRFLOW_HOME: /root/airflow
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@host.docker.internal:5439/airflow
    command: airflow initdb
```

This is the React Application that is in the `redditor-ui` folder and is hosted at `localhost:3000`. 

```
  userinterface:
    image: redditorui
    restart: always
    volumes:
      - ./.env:/usr/src/app/.env
      - ./redditor-ui/public:/usr/src/app/public
      - ./redditor-ui/src:/usr/src/app/src
    environment:
      REACT_APP_HOST: 127.0.0.1
    ports:
      - "3000:3000"
    links:
      - "web:redditapi"
```


This is where all our Airflow dags live. R files are executed with a Bash Executor using the `Rscript` command. If you 
look in `./airflower/scripts/R/shell` you will see where the bash commands live. Let's take a look at one. This 
command `cd`'s into the `r_files` foler and runs the `streamall.R` R script. 


The command:
`. ./airflower/scripts/R/shell/streamall`

executes the following. 
```                                                                                                       
#!/bin/bash

cd /home/scripts/R/r_files
/usr/bin/Rscript /home/scripts/R/r_files/streamall.R
```

You can see the mapping of files in the project to the containers by the following - the left side is the file location
in the project directory and the ride side is in the container. 

`- ./airflower/scripts/R/shell/streamall:/home/scripts/R/shell/streamall`

Anyways, this kicks off the file here at `/home/scripts/R/r_files/streamall.R` which begins to grab Reddit data 
continuously. We see more environment variables we need to have. If you haven't already, go and get 
[Reddit API credentials](`https://www.reddit.com/wiki/api`).

The `sns_send_message` function is from the `biggr` package, which sends a message to a phone number. This requires
access to AWS with IAM + admin privileges. 

These are
```
AWS_ACCESS=YOUR_ACCESS
AWS_SECRET=YOUR_SECRET
AWS_REGION=YOUR_REGION
```

```
library(redditor)
library(biggr)

praw = reticulate::import('praw')

reddit_con = praw$Reddit(client_id=Sys.getenv('REDDIT_CLIENT'),
                         client_secret=Sys.getenv('REDDIT_AUTH'),
                         user_agent=Sys.getenv('USER_AGENT'),
                         username=Sys.getenv('USERNAME'),
                         password=Sys.getenv('PASSWORD'))

sns_send_message(phone_number=Sys.getenv('MY_PHONE'), message='Running gathering')

# Do something with comments
parse_comments_wrapper <- function(x) {
  submission_value <- parse_comments(x)
  write_csv(x = submission_value, path = 'stream.csv', append = TRUE)
  print(now(tzone = 'UTC') - submission_value$created_utc)
}

stream_comments(reddit = reddit_con,
                subreddit =  'all',
                callback =  parse_comments_wrapper)

```


This one manages the state of our dags.
```
  scheduler:
    image: rpy
    restart: always
    depends_on:
      - webserver
    env_file: .env
    environment:
      AIRFLOW_HOME: /root/airflow
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@host.docker.internal:5439/airflow
    volumes:
      - ./airflower/dags:/root/airflow/dags
      - ./airflower/scripts/sql:/home/scripts/sql
      - ./airflower/scripts/R/r_files/aws_configure.R:/home/scripts/R/r_files/aws_configure.R
      - ./airflower/scripts/R/r_files/r_venv_install.R:/home/scripts/R/r_files/r_venv_install.R
      - ./airflower/scripts/R/r_files/refresh_mat_comments_by_second.R:/home/scripts/R/r_files/refresh_mat_comments_by_second.R
      - ./airflower/scripts/R/r_files/refresh_mat_stream_authors.R:/home/scripts/R/r_files/refresh_mat_stream_authors.R
      - ./airflower/scripts/R/r_files/refresh_mat_submissions_by_second.R:/home/scripts/R/r_files/refresh_mat_submissions_by_second.R
      - ./airflower/scripts/R/r_files/stream_submissions_to_s3.R:/home/scripts/R/r_files/stream_submissions_to_s3.R
      - ./airflower/scripts/R/r_files/streamall.R:/home/scripts/R/r_files/streamall.R
      - ./airflower/scripts/R/r_files/streamsubmissions.R:/home/scripts/R/r_files/streamsubmissions.R
      - ./airflower/scripts/R/r_files/streamtos3.R:/home/scripts/R/r_files/streamtos3.R
      - ./airflower/scripts/R/shell/aws_configure:/home/scripts/R/shell/aws_configure
      - ./airflower/scripts/R/shell/refresh_mat_comments_by_second:/home/scripts/R/shell/refresh_mat_comments_by_second
      - ./airflower/scripts/R/shell/refresh_mat_submissions_by_second:/home/scripts/R/shell/refresh_mat_submissions_by_second
      - ./airflower/scripts/R/shell/stream_submissions_all:/home/scripts/R/shell/stream_submissions_all
      - ./airflower/scripts/R/shell/streamtos3:/home/scripts/R/shell/streamtos3
      - ./airflower/scripts/R/shell/r_venv_install:/home/scripts/R/shell/r_venv_install
      - ./airflower/scripts/R/shell/refresh_mat_stream_authors:/home/scripts/R/shell/refresh_mat_stream_authors
      - ./airflower/scripts/R/shell/stream_submission_to_s3:/home/scripts/R/shell/stream_submission_to_s3
      - ./airflower/scripts/R/shell/streamall:/home/scripts/R/shell/streamall
      - ./airflower/plugins:/root/airflow/plugins
      - ./.env:/home/scripts/R/r_files/.Renviron
      - airflow-worker-logs:/root/airflow/logs
    links:
      - "postgres"
    command: airflow scheduler
```

Two R APIS that are exactly the same except for the port location. These are load balanced by the `web` container and the `nginx.conf` file.

Again the `- filelocationlocal:filelocationcontainer` syntax explains how this project is connected.

```
  redditapi:
    image: redditorapi
    command: /app/plumber.R
    restart: always
    ports:
     - "8000:8000"
    working_dir: /app
    volumes:
      - ./plumber.R:/app/plumber.R
      - ./.env:/app/.Renviron
    links:
      - "postgres"
  redditapitwo:
    image: redditorapi
    command: /app/plumber.R
    restart: always
    ports:
     - "8001:8000"
    working_dir: /app
    volumes:
      - ./plumber.R:/app/plumber.R
      - ./.env:/app/.Renviron
    links:
      - "postgres"
```

Where some data is persisted.
```
volumes:
  mongodbdata:
  postgres: {}
  airflow-worker-logs:
```


## THE ENVIRONMENT
My `.env` file looks something like below
```
CHANGE
AIRFLOW__CORE__FERNET_KEY=lasPsWaLsdfoH65nfZfPnggY6O-SrhlQsYBgFf

AWS_ACCESS=MY_ACCESS
AWS_SECRET=MY_SECRET
AWS_REGION=MY_REGION

MONGO_INITDB_ROOT_USERNAME=username # coordinate with init-mongo.js
MONGO_INITDB_ROOT_PASSWORD=password # coordinate with init-mongo.js

POSTGRES_PASSWORD=password

REDDIT_CLIENT=CLIENT ID
REDDIT_AUTH=AUTH
USER_AGENT="datagather by /u/whoeveryouare"
USERNAME=my_reddit_username
PASSWORD=my_reddit_password

MY_PHONE=1-555-555-5555

## DONT CHANGE
POSTGRES_USER=postgres
POSTGRES_DB=postgres
POSTGRES_PORT=5432
POSTGRES_HOST=postgres

AIRFLOW_USER=airflow
AIRFLOW_PASSWORD=airflow
AIRFLOW_DB=airflow


MONGO_INITDB_DATABASE=admin

REACT_APP_HOST=127.0.0.1
PORT=3000

RETICULATE_PYTHON=/root/.virtualenvs/redditor/bin/python

```


Before we kick it off, we need to run these.
```
docker-compose up -d --build postgres
docker-compose up -d --build initdb
```

Now we can start and if we want to watch then run this
```
docker-compose up 
```

We can detach the process by running the detach command
```
docker-compose up -d
```

We can test by hopping into the containers with the commands - 

```
docker exec -it  redditor_scheduler_1  /bin/bash
docker exec -it  redditor_postgres_1  /bin/bash
docker exec -it  redditor_userinterface_1  /bin/bash
```
dc
# Backing Up Your Data

```
psql -U airflow postgres < postgres.bak
```

```
scp -i "~/ndexr.pem" ubuntu@ndexr.com:/var/lib/postgresql/postgres/backups/postgres.bak postgres.bak
docker exec redditor_postgres_1 pg_restore -U airflow -d postgres /postgres.bak
```


# Dont run these unless you know what you are doing. Im serious.
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
docker volume prune
docker volume rm  redditor_postgres_data
```



```
library(mongolite)

mongo_connect <- function(userName, password, collection, database, hostName) {
  
  # m is the default mongo_connection value.
  # i.e., m$find(), m$aggregate()
  try_connection <- function(userName, password, hostName) {
    url  <-
      paste0("mongodb://", userName, ":", password, "@", hostName, ":27017/admin")
    m  <-
      try(mongo(collection = collection , db = database, url = url), silent = TRUE)
    m
  }
  
  m = try_connection(userName, password, hostName[1])
  
  if("try-error" %in% class(m)) {
    m = try_connection(userName, password, hostName[2])
  }
  
  expected_class <-
    c("mongo", "jeroen", "environment")
  
  if(!all(class(m) == expected_class)) {
    stop("No Connection to Mongo available. Check mongo_connect R function.")
  }
  
  m
  
}

m <- 
  mongo_connect(userName = 'fdrennan', 
              password = 'thirdday1', 
              collection = 'data', 
              database = 'admin', 
              hostName = '127.0.0.1')

```



docker exec -it   redditor_scheduler_1  /bin/bash
docker exec -it   redditor_backup_1  /bin/bash
docker exec -it   redditor_redditapi_1  /bin/bash
docker exec -it   redditor_postgres  /bin/bash


pg_dump -h db -p 5432 -Fc -o -U postgres postgres > postgres.bak
wget https://redditor-dumps.s3.us-east-2.amazonaws.com/postgres.tar.gz
tar -xzvf postgres.tar.gz




docker exec -it   airflow_scheduler_1  /bin/bash
## Restore Database
Run Gathering Dag
```
docker exec -it   redditor_postgres  /bin/bash 
tar -zxvf /data/postgres.tar.gz
pg_restore --clean --verbose -U postgres -d postgres /postgres.bak
# /var/lib/postgresql/data
```

psql -U postgres postgres < 



ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo apt update -y'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo apt install apt-transport-https ca-certificates curl software-properties-common -y'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo apt update'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo apt install docker-ce -y'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo usermod -aG docker ubuntu' 


ssh -i "redditor.pem" ubuntu@ndexr.com 'sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'sudo chmod +x /usr/local/bin/docker-compose' 
ssh -i "ndexr.pem" ubuntu@ndexr.com 'git clone https://github.com/fdrennan/ndexr-platform.git'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'cd ndexr-platform && docker build -t redditorapi --file ./DockerfileApi .'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'cd ndexr-platform && docker build -t redditorapi --file ./DockerfileRpy .'
ssh -i "ndexr.pem" ubuntu@ndexr.com 'cd ndexr-platform && docker build -t redditorapi --file ./DockerfileUi .'
 

# To Kill a port
sudo fuser -k -n tcp 3000
sudo fuser -k -n tcp 8000
sudo fuser -k -n tcp 8001
sudo fuser -k -n tcp 8002
sudo fuser -k -n tcp 8003
sudo fuser -k -n tcp 8004
sudo fuser -k -n tcp 8005

rm -rf ~/.ssh/known_hosts
sudo pkill -3 autossh
nohup autossh -M 33201 -N -f -i "~/ndexr.pem" -R -R 3000:localhost:3000 -R 8000:localhost:8000 -R 8001:localhost:8001 -R 8002:localhost:8002 -R 8003:localhost:8003 -R 8004:localhost:8004  ec2-user@ndexr.com &


autossh -i /home/fdrennan/ndexr.pem -R 8999:localhost:8999 -R 3000:localhost:3000 -R 8000:localhost:8000 -R 8001:localhost:8001 -R 8002:localhost:8002 -R 8003:localhost:8003 -R 8004:localhost:8004 ubuntu@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
autossh -f -nNT -i /home/fdrennan/ndexr.pem 9200:localhost:9200 ec2-user@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
autossh -f -nNT -i "~/ndexr.pem" -R 5432:localhost:5432 ec2-user@ndexr.com
autossh  -f -nNT  -i /home/fdrennan/ndexr.pem -R 9200:localhost:9200 ec2-user@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no
autossh -M 20000 -i "ndexr.pem" -f -N ec2-user@ndexr.com -R 3000:localhost:3000 -C
ps aux | grep ssh
sudo kill -9 3082
sudo kill -9 3081
sudo kill -9 3086
sudo kill -9 2982
# UPDATING PORTS
sudo systemctl restart ssh
sudo vim /etc/ssh/sshd_config
/var/log/secure
AllowTcpForwarding yes
GatewayPorts yes

### LENOVO
autossh -f -nNT -i /home/fdrennan/ndexr.pem -R 8999:localhost:8999 -R 3000:localhost:3000 -R 8000:localhost:8000 -R 8001:localhost:8001 -R 8002:localhost:8002 -R 8003:localhost:8003 -R 8004:localhost:8004 ubuntu@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes

### DELL XPS
autossh -f -nNT -i /home/fdrennan/ndexr.pem -R 8081:localhost:8081 ubuntu@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes
autossh -f -nNT -i /home/fdrennan/ndexr.pem -R 9200:localhost:9200 ubuntu@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes

### POWEREDGE
autossh -f -nNT -i /home/fdrennan/ndexr.pem -R 8787:localhost:8787 -R 5433:localhost:5432 ubuntu@ndexr.com -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o ExitOnForwardFailure=yes




# Uploading to Docker
docker image tag rpy:latest fdrennan/rpy:latest
docker push fdrennan/rpy:latest

docker image tag redditorapi:latest fdrennan/redditorapi:latest
docker push fdrennan/redditorapi:latest

# Check open ports
https://gf.dev/port-scanner

# Specify Docker Compose Location
docker-compose -f /Users/fdrennan/redditor/do.yml up

https://analytics.google.com/analytics/web/#/

# Reset Life
```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -f status=exited -q)
docker rmi $(docker images -a -q)
docker volume prune
```

# Install Elastic Search Plugins
https://serverfault.com/questions/973325/how-to-install-elasticsearch-plugins-with-docker-container
