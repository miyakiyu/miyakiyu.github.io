---
title: Simple request searching
date: 2024-06-28 13:31:59
tags: 
- Docker
- MySQL
- Nginx
- Golang
top: 98
---
![Structure chart](/pic/pj.png)
*Using Docker to containerize the MySQL, Nginx and Golang to provide the web service, and managing these containers with docker-compose to create a simple search service.*

<!--more-->

This will be a small application where users can request data through a website. For the web service, I will use Nginx for reverse proxy to Golang (basiclly I like Gin :P), and for the database, I will use MySQL.
This is my folder tree:
```bash
.
├── docker-compose.yml
├── Go
│   ├── Dockerfile
│   ├── main.go
│   └── models
│       └── cat.go
├── MySQL
│   ├── Dockerfile
│   └── my.cnf
└── Nginx
    ├── Dockerfile
    └── nginx.conf
```
Visit my GitHub for more codes: [Simple searching](https://github.com/miyakiyu/DevOps-Exercise/tree/main/Simple%20searching)

So let's start!

---
## Docker
1. ### MySQL settings
First, let's setting mysql image.  
[reference](https://ithelp.ithome.com.tw/articles/10340991)
``` bash
# MySQL setting file
FROM mysql:latest

# Setting mysql
ENV MYSQL_DATABASE=(Your db)
ENV MYSQL_USER=(Your user name)
ENV MYSQL_PASSWORD=(Your password)
ENV MYSQL_ROOT_PASSWORD=(Your password)

# Using my.cnf
COPY my.cnf /etc/my.cnf

# Set workdir
WORKDIR /var/lib/mysql

# Set port, deafult is 3306 or anyother you like
EXPOSE 3306

# Start service
CMD ["mysqld"]
```
As you can see I define a  file call "my.cnf", it's a config file for mysql.
``` bash
[mysqld]
# If general_log is set to 1, MySQL will log all queries
general_log = 1

# Specify the path for storing the general log
general_log_file = /var/log/mysql.log

# Log output will be written to a file
log_output = file
```
After finishing the Dockerfile, you can test it first. Run the following command to build an image:
`docker build --no-cache -t (Your image name) .`

After building the image, you can run it with:
`docker run --name (Give it a name) -p 3306:3306 (Your image name)`

And test mysql service is on, using this:
`mysql -u root -p -h 127.0.0.1 -P 3306`
After that you should now see the service running successfully!

If you want to see the logs, you can enter the container:
`docker exec -it (Your container name) /bin/bash`

2. ### Nginx settings
Created Dockerfile like this:

``` bash
# Nginx settings
FROM nginx:latest

# Setting nginx
# Copy the local nginx.conf to Docker 
COPY nginx.conf /etc/nginx/nginx.conf

# Open the port 80 for Using 
EXPOSE 80
```
After we done the Dockerfile setting then we could start setting the Nginx.
``` bash
events{
  worker_connections 1024;
}

http {
  server{
    listen 80;
    server_name localhost;
    # How to handle the request from /
    # We rever proxy to backend(Golang service) port 8080
    location / {
      proxy_pass http://backend:8080;
    }
  }
}
```
So now we have the reverse proxy to Golang.

3. ### Golang settings
Now let's setting Golang

```bash
FROM golang:latest

WORKDIR /app

# Copy all file into image
COPY . .

# The hardest part is the Go module :(
RUN go mod init main
RUN go mod tidy
# DO NOT RUN GO APP DIRECTLY! It won't stop :(
RUN go build -o main

# Different then Nginx
EXPOSE 8080

CMD ["./main"]
```

And here I choose using Gin for interacting with MySQL and return the result.
So here I using official example for test reverse proxy is fine.

---
## Gin with MySQL
1. ### MySQL pre-set
First, we create a new database:
```sql
CREATE DATABASE cat;
```
And we check our database:
```sql
SHOW databases;
```
```sql
+--------------------+
| Database           |
+--------------------+
| cat                |
| information_schema |
| mydatabase         |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
We use this database for create a new table inside of it, for store data:
```sql
USE cat; (If you suceesefully change database you will see 'Database changed')
```
```sql
CREATE TABLE cats (`id` int, `name` varchar(255));
```
So far we can use this command to check if the table is sucessfully create:
```sql
DESCRIBE cats;
```
And you should see:
```sql
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| id    | int          | YES  |     | NULL    |       |
| name  | varchar(255) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
```
Now we insert some value into table, and check it:
```sql
SELECT * FROM cats;
```
This is our final table looks like:
```sql
+------+----------+
| id   | name     |
+------+----------+
|    1 | Nekoking |
|    2 | Neko1    |
|    3 | Neko2    |
+------+----------+
```

2. ### Gin 
In last part we create database and table, now we use Gin to interate with MySQL, here we only doing searching.
Our folder tree lioos like:
```bash
.
├── Dockerfile
├── main.go
└── models
    └── cat.go
```
Let's looks "main.go"
``` golang
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"

	"(Your folder)/models"
)

func main() {
	// Globla setting
	dsn := "YOUR_USER_NAME:YOUR_PASSWORD@tcp(YOUR_IP_AND_PORT)/YOUR_DATABASE"
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalf("failed to connect to database: %v", err)
	}

	r := gin.Default()
	// Home 
	r.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"data": "Hello World"})
	})

	// Get cats
	r.GET("/cats", func(c *gin.Context) {
		cats, err := models.GetCats(db)
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.JSON(http.StatusOK, cats)
	})

	r.Run()
}
```
I use GORM to interactive with MySQL database, and here is my /modles:
```golang
package models

import (
	"gorm.io/gorm"
)

type Cat struct {
	ID   uint   `json:"id"`
	Name string `json:"name"`
}

func GetCats(db *gorm.DB) ([]Cat, error) {
	var cats []Cat
	result := db.Find(&cats)
	return cats, result.Error
}
```
Define struct for store result.

---
## Problems Fixing
When I doing this project I have be beaten by a lot of unexcept problems, I write it here for those who have the same problem! :D

### Can't connect MySQL with GORM
[reference](https://medium.com/tech-learn-share/docker-mysql-access-denied-for-user-172-17-0-1-using-password-yes-c5eadad582d3)
![Error](/pic/problem.png)
I've been trying to use GORM to connect to MySQL, but I've been encountering consistent failures. It's frustrating and waste my time. :(
I suspected the issue might be due to incorrect user credentials or IP settings. So, I accessed the MySQL container to create a new user with the correct permissions and IP. After that, I finally able to connect successfully!





