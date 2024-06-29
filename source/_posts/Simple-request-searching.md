---
title: Simple request searching
date: 2024-06-28 13:31:59
tags: 
- Kubernetes 
- Docker
top: 98
---
![Structure chart](/pic/side_project.jpg)
*Using Docker to containerize the MySQL service and Nginx to provide the web service, and managing these containers with Kubernetes to create a simple search service.*

<!--more-->

I decided to use K8S to control and manage my Docker containers. This will be a small application where users can request data through a website. For the web service, I will use Nginx, and for the database, I will use MySQL.
So let's start!

---
## Docker
1. ### Create Nginx and MySQL container

