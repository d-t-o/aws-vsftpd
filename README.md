# aws-vsftpd
A quick and cheap way to host an FTP Server (vsftpd) on AWS.

The idea for this started when I bought a Xiaomi Dafang. That's a home surveillance camera with an [open source firmware](https://github.com/EliasKotlyar/Xiaomi-Dafang-Hacks) that I activate when I leave my apartment for a longer time, e.g. during vacation. It will send a notification via mail and record a video once motion is detected. 

As it makes no sense to have a video of someone breaking into your home recorded on a local device that can just be taken, this solution needs some remote storage. In an ideal world we would just write to amazons S3 directly as it's (relatively) cheap online storage but sadly there's no S3 support built in to this device and it's also not very powerful either, so installing python and awscli is no option.

The intended way of uploading your data to a remote location is via FTP. So I needed an FTP server that I could start on demand, run for as long as needed while having minimal cost. Plus I don't want to configure stuff manually, so it had to be more or less a click and forget solution.

This is when this project came to life, as it does just that. There was just one problem: if you start a machine on aws, you have to decide how much storage you need. The more storage you need, the more expensive it gets. But I only need storage for the uploaded video and how much video is uploaded depends on how much movement the burglars make while robbing me. So instead of reserving fixed space upfront, I just wanted to write to S3 directly as that is amazons way of having unlimited space while paying just for the fraction you use.

So the idea is to:

* Build a docker image of vsftpd
* Put it on amazons ECR
* Start the cheapest EC2 instance possible
* Mount an S3 bucket as docker volume
* Start the docker container with the FTP folder mapped into the S3 bucket

Doing this allows me to use a t3a.nano instance, which at the moment of writing this costs in my region <1 usd per week. Which is rather cheap for your own FTP server with an unlimited storage option.

# What do you need to host your FTP server on AWS?

1. An [aws account](https://aws.amazon.com). It's (at the time of writing this) free and comes with free contingent of all kinds of services during the first year.
2. The ability to build a docker image on your computer. If you're new to this you want to look for a docker setup tutorial for your OS.
3. Optionally: a free account for [duckdns](https://duckdns.org). If you have one you'll get a static name for your FTP server instead of an IP address, which allows you to have a fixed configuration on the device that writes to the FTP server.

# Usage
This repository has two folders, one for the docker image of vsftpd, one for the aws stack. Building the docker container is technically not necessary as it could also be taken pre-built from a repository somewhere, but for some reason it feels weird to me to start a container some private person from the internet has built and claims does what he says. Building it takes like 10 seconds if you have your docker environment setup already.

## 1. Setup the Docker Repository Stack
1. Login to your aws account, select the correct region on the top right, otherwise your server could end up on the wrong continent.
2. Select *CloudFormation* from the *Services* menu.
3. Go to *Stacks*, select *Create Stack* and *with new resources*.
4. Select *upload a template file* and the *01_ftp.yaml* file from the aws folder for upload.
5. On the next page enter a name for the stack, e.g. 'dafang-ftp-1' and choose a prefix for your S3 bucket. Make sure it's unique enough it doesn't clash with somebody elses bucket. We'll use *dafang-your_initials* for this sample, which is probably ok. Click *Next* until there's no Next anymore.

You should now have a stack in a green status. 

## 2. Build the Docker Image

The docker config is mainly taken from [here](https://github.com/delfer/docker-alpine-ftp-server). The difference is it uses the latest alpine linux instead of a fixed version and an issue in the startup script was fixed that would let the vsftpd process exit prematurely.

* Open a command prompt on your computer in the docker folder and execute ```docker build -t vsftpd .```
* On the aws website, select *Services*, type *ECR* and select *Elastic Container Registry*. Follow the link and you should see one repository called *vsftpd*. Select it.
* Click on *View push commands* top right and follow the instructions.

You should now see an image tagged *latest* in the images list.

## 3. Launch the FTP server
* On the aws website select *CloudFormation* from the *Services* menu again.
* Go to *Stacks*, select *Create Stack* and *with new resources*.
* Select *upload a template file* and the *02_ftp.yaml* file from the aws folder for upload.
* On the next page enter a name for the stack, e.g. *dafang-ftp-2*.
* Fill in the mandatory parameters. 

When I started with this project, I used the same mechanism I use at work to make sure server instances are up and running: start them through an auto scaling group that monitors their availability and whenever a server is not running or answering anymore, start a new one. This is a nice, stable solution but it has one major downside: the load balancer used costs more per hour than the FTP server. Since I wanted a cheap solution, I made this step optional, so you can ignore the *MonitorEc2Instance* part, it's most likely not what you want.

What can be confusing to new aws users is probably the ami id you have to specify. That identifies the OS image that amazon will put on the server instance you're about to start. This image id is different in every region and also changes often, whenever amazon builds a new release of their linux. So to get the latest ami id go to *Services* -> *EC2* -> *Launch Instance*. There you'll get a long list of available images in your region. The amazon linux 2 x86 one is usually the topmost. 

* Click *Next* until there is no Next anymore.

You should now have another stack in a green status. Select that stack, click on *Outputs* and there's your FTP server url. If you go to *Services* -> *S3* you will see your bucket and can also download files from there. Whatever was uploaded to the FTP server before should be visible in there.

## How to stop all this?
* On the aws website select *CloudFormation* from the *Services* menu.
* Select the second stack *dafang-ftp-2*
* Click on *Delete*, confirm and wait a few seconds.

The stack should be deleted. From this moment on, you only pay for the FTP data still in your S3 bucket plus a super low fee for the docker image still being in the ECR. 

If you also want to get rid of that, also delete the first stack *dafang-ftp-1*. You will have to delete the docker image from the ECR manually though as aws refuses to delete some resources that still contain user data. Also you'll have to manually delete the S3 bucket with the FTP data in it for the same reason.