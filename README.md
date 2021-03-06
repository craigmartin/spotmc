[![Circle CI](https://circleci.com/gh/goura/spotmc.svg?style=svg)](https://circleci.com/gh/goura/spotmc)

SpotMC
=======
SpotMC is a utility to host a craftbukkit/minecraft_server type game on an AWS EC2 spot instance.

Using SpotMC, you might be able to:
* Start your game server on a spot instance, which is very cheap in price
* Save your game state when terminating a spot instance for future play
* Automatically terminate your spot instance after a certain idle time so that you won't get billed a lot just because you forgot to terminate it
* Set a hard-limit of an uptime so that you don't have to worry about unexpected bills

It is aimed to let you host a craftbukkit/minecraft server to play with your family, friends.

Status
=======
I've been running a minecraft server on a spot instance for my family
for a year using adhoc shell scripts to autoscale and auto-terminate,
and I'm now trying to port it into a more solid something (to learn Go).

It's still in its early development process.
Please use this utility at your own risk.

What You Have to Prepare
=========================
- Access to AWS Management Console. You are going to setup an AutoScaling group. Game state will be saved in S3.
- Game server jar file (like craftbukkit.jar/minecraft_server.jar) and its eula.txt (for minecraft). It's not included in this software.
- Recommended: Dynamic DNS(DDNS) update URL. Your server will get a different IP address everytime it launches, so you should want a way to fix its host name. DDNS is good for that purpose. DDNS providers often provides an "update URL" to let the server set the global IP of itself. SpotMC has an ability to make use of that URL. Take note of that URL.

Set Up Steps (using CloudFormation)
=====================================
If you have a Default VPC in your AWS account you can use the CloudFormation template.

1. Put the game server jar file and eula file on S3
2. Create an IAM role which allows access to S3 and EC2 (narrow down the grant as you like)
3. Choose "Create Stack" in  CloudFormation control panel, and load `cfm-template.json` as a template
4. Fill in the parameters and create a stack, wait until it completes... you're done!
5. Edit the Auto Scaling Group and set the number of instances of Desired to 1, to launch a server, and wait until the server starts
6. When you want to shut down the instance, set Desired to 0 (or it should be automatically shut down after a while if no one is logging in to the game)


Manual Set Up
==============

Sample User Data 
-------------------
```
#!/bin/bash
export SPOTMC_DDNS_UPDATE_URL='http://XXXXXXXX:XXXXXXX@dynupdate.no-ip.com/nic/update?hostname=XXXXXXXX.no-ip.org'
export SPOTMC_SERVER_JAR_URL=s3://XXXXXXXX/minecraft_server.1.8.1.jar
export SPOTMC_SERVER_EULA_URL=s3://XXXXXXXX/eula.txt
export SPOTMC_DATA_URL=s3://XXXXXXXX/data.tgz
export SPOTMC_JAVA_PATH=/usr/bin/java
export SPOTMC_JAVA_ARGS="-Xmx1024M -Xms1024M"
export SPOTMC_AWS_REGION="ap-northeast-1"
export SPOTMC_KILL_INSTANCE_MODE="shutdown"

# Download spotmc
cd /
wget https://github.com/goura/spotmc/releases/download/0.0.1/spotmc
chmod 755 spotmc

# Set a dummy initscript
# This prevents sudden shutdown and spares time to sync data to S3.
./spotmc -rhinitscript > /etc/init.d/dummy-smc-stopper
chmod 755 /etc/init.d/dummy-smc-stopper
chkconfig --add dummy-smc-stopper
/etc/init.d/dummy-smc-stopper start

# Run spotmc
nohup ./spotmc &
```


Parameters
------------

spotmc command uses env vars for configuration

* `SPOTMC_SERVER_JAR_URL` (mandatory)
    * Specify the URL of the game server jar in `s3://{bucket}/{key}` format

* `SPOTMC_SERVER_EULA_URL` (mandatory)
    * Specify the URL of the eula.txt file in `s3://{bucket}/{key}` format

* `SPOTMC_DATA_URL` (mandatory)
    * Specify the path where you like to save the data in `s3://{bucket}/{key}` format. Currently spotmc saves the data as a single tgz file.

* `SPOTMC_JAVA_PATH` (mandatory)
    * Specify the full path to java cmd (like `/usr/bin/java`).

* `SPOTMC_AWS_REGION` (default="ap-northeast-1")
    * Which AWS region to host the server in

* `SPOTMC_JAVA_ARGS` (default=none)
    * Extra args to give to java cmd, like `-Xmx1024M -Xms1024M`

* `SPOTMC_MAX_IDLE_TIME` (default=14400)
    * The time which after everyone logs out from the server, spotmc tries to terminate the instance. Specify this in seconds.

* `SPOTMC_MAX_UPTIME` (default=43200)
    * The time after which no matter whether someone is still playing or not, the server will terminate. Specify this in seconds.

* `SPOTMC_IDLE_WATCH_PATH` (default="world/playerdata")
    * The directory, relative to game data root, to watch for the game activity. If the specified path is inactive (i.e. doesn't get updated) for `SPOTMC_MAX_IDLE_TIME`, spotmc tries to shutdown the autoscaling group which the instance is belonging to.

* `SPOTMC_DDNS_UPDATE_URL` (default=none)
    * When spotmc starts, it accesses this URL. Use it to update your DDNS settings.
    * If this parameter is not specified, spotmc won't do anything.

* `SPOTMC_KILL_INSTANCE_MODE` (default="false")
    * spotmc tries to kill the instance when the game server goes down for some reason, or when it detected the spot instance termination notification
    * When this parameter is set to "false" it will not actually shutdown the instance. This is just for safety not to casually kill your server.
    * When this parameter is set to "shutdown" it will call `SPOTMC_SHUTDOWN_CMD` to kill the instance. This is a recommended parameter.

* `SPOTMC_SHUTDOWN_CMD` (default="/sbin/shutdown -h now")
    * The command called when spotmc is killing the instance
