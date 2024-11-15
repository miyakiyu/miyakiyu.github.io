---
title: Jenkins with GitHub
date: 2024-08-23 12:16:54
tags:
- Jenkins
- Github
- ngrok
top: 97
---
*Using Jenkins to deploy a job, active Github webhook, when we push to Github then Jenkins will automatic doing something*
In this article I will using Jenkins to made a automatic job that is when we push new commit on Github, then it will automatically to doing something you setting. I though it's a good way to practicing CI.
<!--more-->
## Overall
We need to set up a public domain for GitHub to access our Jenkins service. We can use Ngrok for this purpose. Ngrok acts like a post office: when GitHub sends a request to Jenkins, it can't find my IP address directly. However, by using Ngrok, the request is sent to Ngrok first, and then Ngrok forwards the request to the localhost, making it appear as if we have a public IP address.
## Ngrok 
![ngrok](/pic/jenkins/1.png)
Setting Ngrok, let GitHub webhook to found your Jenkins service.
## Github
![webhook](/pic/jenkins/2.png)
Setting Github webhook for connectin to the Jenkins servicve.
## Jenkins
First, create a new job, and setting the git repo URL for which you want to following. Then setting the action that when Jenkins notice the Github is pushing new commit then what you want to do and save it.
And do a test push.
![jenkins](/pic/jenkins/3.png)
