---
title : "Application"
date : "`r Sys.Date()`"
weight : 3
chapter : false
pre : " <b> 3. </b> "
--- 

# Overview
This is a simple web application written in Golang that simulates some basic functionalities of a banking application (account management, money transfer).
Source code: [bank01](https://github.com/pcuser24/bank01.git)

# Functionalities
The application has 2 routes:
1. `/users` - This route handle POST requests to create a new user account. Each user account has a unique ID, name, and avatar. On successful creation, the avatar is stored in a S3 bucket and the new user account is stored in a RDS PostgreSQL database.
2. `/healthcheck` - This route handle GET requests to check the health of the application. On occastion of success, it returns a simple text `pong-<ip address of the host>` with the status code 200. 
3. Other routes handle requests related to money transfer between user accounts, but not considered in this workshop.
