# Lab 5: Auto Scaling Your EC2 Instances

**Status:** ✅ Complete

**Date Completed:** May 4, 2026

**Reference:** [AWS Network Challenge 2 by Raphael Jambalos](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)

---

## 🔹 Overview

Lab 4 made the App Server stateless. Lab 5 is where that decision pays off. With all data living in RDS, DocumentDB, and EFS, the App Server itself holds nothing. That means it can be cloned. And if it can be cloned, it can be cloned automatically.

Lab 5 introduces Auto Scaling Groups. Instead of manually adding servers when traffic spikes, the ASG watches CPU utilization across all running instances and makes the decision itself. When the average crosses 70%, it launches a new identical copy of the App Server. When the load drops, it terminates the extra instance. The ALB from Lab 3 is already in place to distribute traffic across however many instances are running at any given time. None of this requires any manual action after the initial setup.

The lab involved four main pieces: creating an AMI from the current App Server as a blueprint, building a Launch Template that tells the ASG exactly how to start new instances, creating the Auto Scaling Group itself with the correct capacity limits and subnet configuration, and verifying the whole system by running a CPU stress test that actually triggered a scale-out event.

![Lab 5 Architecture](images/lab5/lab5-architecture-diagram.png)

*Source: [Raphael Jambalos — AWS Network Challenge 2](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)*

---

## 🔹 Goal

Build an Auto Scaling Group that can automatically add and remove App Server instances based on CPU load:

- Create a startup script so new instances can start Flask without any manual intervention
- Capture the current App Server state as an AMI
- Build a Launch Template using that AMI with a User Data script
- Create an Auto Scaling Group connected to the existing ALB target group
- Configure a dynamic scaling policy triggered by CPU utilization
- Verify scale-out and scale-in with a live CPU stress test

---

## 🔹 What I Built

**AWS Resources Created:**

- 1 startup script (`/home/ec2-user/start-flask.sh`) baked into the AMI
- 1 environment variable file (`/home/ec2-user/flask-env.sh`) baked into the AMI
- 1 Amazon Machine Image (`flask-app-server-ami`)
- 1 Launch Template (`flask-app-launch-template`) with User Data
- 1 Auto Scaling Group (`flask-app-asg`) with min 1, desired 1, max 3
- 1 Dynamic Scaling Policy (`flask-app-cpu-scaling-policy`) at 70% CPU
- 2 CloudWatch Alarms (auto-created by the scaling policy)

**Configuration:**

- ASG connected to `flask-app-target-group`
- ASG restricted to `flask-app-private-subnet` (`ap-southeast-1a`) only
- Health check grace period: 120 seconds
- Instance warmup: 120 seconds
- Original `flask-app-application-server` deregistered from target group and stopped

---

## 🔹 Code Integration

The Flask application (`main.py`) was not modified. The environment variables that connect Flask to RDS, DocumentDB, and EFS are unchanged from Lab 4. The only new code is the two shell scripts created on the server before the AMI was taken.

`/home/ec2-user/flask-env.sh` stores all eight environment variables:

```bash
#!/bin/bash
export UPLOAD_DIRECTORY='/home/ec2-user/efs/uploads'
export MONGODB_DB_CONNECTION_URI='mongodb://flaskadmin:PASSWORD@YOUR_DOCDB_ENDPOINT:27017/'
export MONGODB_DB_NAME='ecv-jmp-file-upload-app'
export ENV_MODE='frontend'
export POSTGRESQL_DB_HOST='YOUR_RDS_ENDPOINT'
export POSTGRESQL_DB_DATABASE_NAME='ecv_file_upload_app_psql'
export POSTGRESQL_DB_USERNAME='flask_app_admin'
export POSTGRESQL_DB_PASSWORD='YOUR_RDS_PASSWORD'
```

`/home/ec2-user/start-flask.sh` sources those variables and starts Flask:

```bash
#!/bin/bash
source /home/ec2-user/flask-env.sh
cd /home/ec2-user/file-upload-flask
source venv/bin/activate
flask --app main run --host 0.0.0.0 >> /home/ec2-user/flask.log 2>&1
```

The User Data script in the Launch Template calls `start-flask.sh` on every new instance's first boot:

```bash
#!/bin/bash
sleep 15
sudo chown -R ec2-user:ec2-user /home/ec2-user/efs
mkdir -p /home/ec2-user/efs/uploads
sudo -u ec2-user bash /home/ec2-user/start-flask.sh &
```

**Environment variables — no changes from Lab 4:**

| Variable | Lab 4 Value | Lab 5 Value |
|---|---|---|
| `UPLOAD_DIRECTORY` | `/home/ec2-user/efs/uploads` | Unchanged |
| All others | Lab 3 managed service endpoints | Unchanged |

---

## 🔹 My Experience

### Preparing the Server Before the AMI

Before taking the AMI, I needed to solve a problem that is easy to overlook: environment variables. In Labs 3 and 4, I set environment variables manually inside a tmux session every time I SSHed into the server. That approach works when there is one server and one person managing it. It breaks completely when the ASG launches a new instance at 3am during a traffic spike with nobody there to type export commands.

The fix was to create two files that get baked into the AMI and run automatically on every boot. The first file, `flask-env.sh`, stores all eight environment variables. The second file, `start-flask.sh`, sources that file, activates the Python virtual environment, and starts Flask. Every new ASG instance will have these files from the moment it launches.

After creating `start-flask.sh` and making it executable, I tested it manually by running it in the background and then checking with curl:

![Startup Script Working](images/lab5/lab5-01-startupscript-working.png)

*`curl http://localhost:5000` returning the expected response after running `start-flask.sh` directly, confirming the script works before baking it into the AMI*

---

### The Missing start-flask.sh

The first thing that went wrong was simple but worth documenting. I created `flask-env.sh`, made it executable, and jumped straight to running `start-flask.sh` without ever creating it. The terminal returned:

```bash
-bash: /home/ec2-user/start-flask.sh: No such file or directory
```

![start-flask.sh Missing](images/lab5/lab5-troubleshoot-01-startflask-missing.png)

*Running `/home/ec2-user/start-flask.sh` before the file was created, resulting in a "No such file or directory" error*

The fix was straightforward. I created the file with nano, pasted the correct content, made it executable, and ran it again successfully. The lesson here is that the two scripts serve different purposes and both need to exist before the AMI is taken.

---

### Creating the AMI

With both scripts in place and tested, I stopped Flask cleanly and took the AMI from the AWS Console.

The AMI took a few minutes to move from pending to available.

![AMI Available](images/lab5/lab5-02-ami-available.png)

*`flask-app-server-ami` showing Available status in the AMI console*

An important side effect of the reboot: the tmux session where Flask was running was killed. After the AMI was created, I SSHed back into the App Server and found no tmux sessions. I created a new one and restarted Flask manually so the original server would keep serving traffic while the ASG was being configured.

---

### Creating the Launch Template

The Launch Template is the configuration layer on top of the AMI. The AMI captures what is installed. The Launch Template captures how to run it. The most important part of the Launch Template is the User Data script, which runs automatically on every new instance's first boot.

```bash
#!/bin/bash
# Wait for EFS to be mounted via fstab
sleep 15

# Set ownership of EFS mount in case it's a fresh mount
sudo chown -R ec2-user:ec2-user /home/ec2-user/efs

# Create uploads directory if it doesn't exist yet
mkdir -p /home/ec2-user/efs/uploads

# Start Flask as ec2-user
sudo -u ec2-user bash /home/ec2-user/start-flask.sh &
```

The User Data script does three things: waits 15 seconds for EFS to mount via fstab, ensures the EFS directory is owned by `ec2-user`, and runs `start-flask.sh` as `ec2-user` in the background. The `sudo -u ec2-user` part is critical because User Data runs as root, and Flask needs to run as `ec2-user` to access the EFS directory with the correct permissions.

![Launch Template Created](images/lab5/lab5-03-launchtemplate-created.png)

*`flask-app-launch-template` created successfully with Version 1*

---

### Creating the Auto Scaling Group

The ASG setup had one important constraint: subnet selection. I selected only `flask-app-private-subnet` (`10.0.2.0/24`, `ap-southeast-1a`) and not the second private subnet. The EFS mount target only exists in `ap-southeast-1a`. An instance launched in `ap-southeast-1b` would fail to mount EFS on boot, Flask would not start, and the ASG would enter a loop of launching and immediately terminating unhealthy instances.

I connected the ASG to `flask-app-target-group`, set the health check grace period to 120 seconds, and configured the group size as Desired 1, Min 1, Max 3.

![ASG Created](images/lab5/lab5-04-asg-created.png)

*`flask-app-asg` created with Desired: 1, Min: 1, Max: 3*

The ASG immediately launched its first instance. I watched the Activity tab and waited for the new instance to appear in the target group as healthy.

![ASG First Launch Activity](images/lab5/lab5-05-asg-activity-firstlaunch.png)

*ASG Activity tab showing the first instance being launched after group creation*

---

### Watching the First ASG Instance Come Up

Once the ASG instance was running and healthy, the target group showed two instances: the original `flask-app-application-server` and the new `flask-app-asg-instance`.

![Two Instances Healthy](images/lab5/lab5-06-twoinstances-healthy.png)

*Target group showing both the original App Server and the new ASG instance as healthy*

I deregistered the original App Server from the target group. From this point forward, all traffic goes through the ASG-managed instance. The original server was stopped but not terminated, kept as a reference in case debugging was needed.

![ASG Only Target](images/lab5/lab5-07-asg-only-target.png)

*Target group after deregistering the original App Server, with only the ASG instance remaining as a target*

---

### Verifying the App Works Through the ASG Instance

Before running the stress test, I verified the app was fully functional through the ASG instance. I visited the ALB DNS name in a browser to confirm the health check route returned correctly, then tested the upload and images routes.

![App Working via ASG](images/lab5/lab5-08-app-working-via-asg.png)

*App returning the expected response via the ALB DNS, now being served by the ASG instance*

![Upload File Working](images/lab5/lab5-09-app-upload-file-working-via-asg.png)

*Upload form accessible through the ALB with the ASG instance serving the request*

![Upload Successful](images/lab5/lab5-10-img-uploaded-successfully-via-asg.png)

*Successful upload response confirming Flask on the ASG instance can write to EFS*

![Images Page Working](images/lab5/lab5-11-app-images-working-via-asg.png)

*Images page loading correctly, confirming the ASG instance can read from EFS*

---

### The Stress Test and the Missing Scaling Policy

This is where the lab took an unexpected turn. After SSHing into the ASG instance at `10.0.2.75` and running the stress test, I waited for the ASG to scale out. Five minutes passed. The stress test expired. Nothing happened in the ASG Activity tab.

The first problem was the CPU monitoring. I opened a second terminal session to check `top` on the same instance, but the first run showed CPU nearly idle at 94% idle. The stress test had already expired by the time I checked. I reran the stress test and this time verified CPU immediately in a second SSH session:

![Stress CPU Idle](images/lab5/lab5-troubleshoot-02-stress-cpu-idle.png)

*`top` showing CPU nearly idle at 94% idle during what should have been an active stress test, indicating the stress had already expired*

![Stress CPU Working](images/lab5/lab5-troubleshoot-03-stress-cpu-working.png)

*`top` showing `%Cpu(s): 98.7 us` with two stress worker processes each consuming ~49.5% CPU, confirming the stress test is running correctly*

But even with CPU confirmed at 99%, the ASG still did not scale out. I went to check the CloudWatch alarms to see if the CPU threshold had been breached. The alarms page was completely empty:

![No CloudWatch Alarms](images/lab5/lab5-troubleshoot-04-no-cloudwatch-alarms.png)

*CloudWatch Alarms showing zero alarms, meaning no alarm existed to trigger the scale-out*

The root cause was that the dynamic scaling policy was never created during the ASG setup. Without a scaling policy, no CloudWatch alarm gets created, and the ASG has no signal to act on regardless of how high CPU climbs.

![No Scaling Policy](images/lab5/lab5-troubleshoot-05-no-scaling-policy.png)

*ASG Automatic Scaling tab showing zero dynamic scaling policies, confirming the scaling policy was never configured*

I went back to the ASG Automatic Scaling tab and created the policy manually: Target tracking, Average CPU Utilization, 70% threshold, 120-second warmup. AWS automatically created two CloudWatch alarms from that policy, one for scale-out and one for scale-in.

![Scaling Policy Created](images/lab5/lab5-18-scaling-policy-created.png)

*`flask-app-cpu-scaling-policy` listed under Dynamic scaling policies after being created manually*

![CloudWatch Alarms Active](images/lab5/lab5-19-cloudwatch-alarms-active.png)

*CloudWatch showing the two alarms automatically created by the scaling policy*

With the policy in place, I ran the stress test again with a longer timeout of 600 seconds to give CloudWatch enough time to collect data points and trigger.

---

### Watching Scale-Out and Scale-In

After the stress test confirmed CPU at 99%, I monitored the ASG Activity tab and CloudWatch. Within a few minutes the alarm state changed to In alarm, and the ASG launched additional instances.

![Scale-Out Activity](images/lab5/lab5-12-asg-scaleout-activity.png)

*ASG Activity tab showing the scale-out event triggered by the CPU alarm*

![Three ASG Instances Running](images/lab5/lab5-13-threeasginstances-running.png)

*EC2 Instances showing three `flask-app-asg-instance` instances running at peak load*

![Three Targets Healthy](images/lab5/lab5-14-threetargets-healthy.png)

*Target group showing all three ASG instances registered and healthy, with the ALB distributing traffic across all of them*

After the stress test expired and CPU dropped back to normal, the ASG began scale-in. The extra instances were terminated, connection draining showed the targets entering draining state before removal, and the ASG settled back to one instance.

![Scale-In Activity](images/lab5/lab5-15-asg-scalein-activity.png)

*ASG Activity tab showing the scale-in event after CPU normalized*

![Two ASG Instances Terminated](images/lab5/lab5-16-twoasginstances-terminated.png)

*EC2 Instances showing the two extra instances terminated after scale-in completed*

![Two Targets Draining](images/lab5/lab5-17-twotargets-draining.png)

*Target group showing the two instances in draining state as the ASG deregistered them cleanly before termination*

---

## 🔹 Final Verification

All verification steps passed. The stress test triggered a real scale-out event, the ASG launched new instances automatically, they registered in the target group and passed health checks, uploads continued to work through the ALB during the scale-out period, and the scale-in removed the extra instances cleanly after CPU normalized. The app remained accessible throughout the entire cycle without any manual intervention.

---

## 🔹 Errors and Fixes Summary

| Error | Cause | Fix |
|---|---|---|
| `/home/ec2-user/start-flask.sh: No such file or directory` | `flask-env.sh` was created but `start-flask.sh` was never created before running it | Created `start-flask.sh` with nano, pasted the correct content, made it executable with `chmod +x`, and reran |
| Stress test running but CPU showing idle in `top` | The stress test had already expired by the time `top` was checked in the second session | Reran the stress test with a longer timeout and verified CPU in a second SSH session immediately |
| ASG never scaled out despite CPU at 99% | The dynamic scaling policy was never created during ASG setup, so no CloudWatch alarm existed to trigger the scale-out | Created the `flask-app-cpu-scaling-policy` manually from the ASG Automatic Scaling tab, which auto-created the two required CloudWatch alarms |

---

## 🔹 Key Learnings

**1. The AMI captures state. The Launch Template captures behavior. The User Data script is the bridge.**

Taking an AMI without a startup script produces a blueprint that knows what software is installed but has no idea what to do with it. The AMI is a frozen server. User Data is what brings it to life on every boot. Without `start-flask.sh` and the User Data script calling it, every ASG instance would launch, pass the OS health check, fail the ALB health check because Flask never started, and get terminated in a loop. The three-part system, AMI plus Launch Template plus startup script, is what makes an ASG instance actually useful.

**2. A scaling policy is not optional configuration. It is the entire point of the ASG.**

The ASG was created correctly in terms of the network, subnet, target group, and capacity settings. But without a dynamic scaling policy, it was just a managed way to keep one instance running. No alarm means no signal. No signal means no scale-out, regardless of how high CPU climbs. The policy is what transforms the ASG from a monitoring tool into an automation tool.

**3. CloudWatch needs time. Stress tests need to outlast the evaluation period.**

CloudWatch collects EC2 metrics once per minute. A target tracking policy needs several consecutive minutes of data above the threshold before the alarm fires. A five-minute stress test can expire before CloudWatch has collected enough data points to act. Running the test with a 600-second timeout gave CloudWatch the window it needed to evaluate the metric, breach the threshold, and trigger the scale-out. Rushing the test is the same as not running it.

**4. The single-AZ subnet constraint from Lab 4 carries forward into Lab 5.**

The EFS mount target only exists in `ap-southeast-1a`. This was a deliberate choice in Lab 4 to avoid the cost of a second mount target. In Lab 5, that constraint must be respected when configuring the ASG subnet. Selecting both private subnets would cause the ASG to occasionally launch instances in `ap-southeast-1b`, where EFS is unreachable. Flask would start, fail to find the uploads directory, and the instance would be marked unhealthy. Choosing only `flask-app-private-subnet` in `ap-southeast-1a` is not just a best practice here. It is a hard requirement imposed by the Lab 4 architecture decision.

**5. Stateless architecture is what makes all of this possible.**

Three ASG instances ran simultaneously during the peak of the stress test. All three served traffic. All three read and wrote to the same EFS directory. All three connected to the same RDS instance and the same DocumentDB cluster. Not one of them held any data of its own. A user could upload an image that hits Instance A, then view the images page on a request routed to Instance B, and the image appears correctly because both instances share the same EFS volume. This only works because of everything built in Labs 3 and 4. The auto-scaling in Lab 5 is the payoff for making the server stateless first.

---

## 🔹 Cleanup Performed

| Action | Reason |
|---|---|
| Deregistered original `flask-app-application-server` from target group | ASG now manages all instances. The original server no longer needs to receive traffic |
| Stopped original `flask-app-application-server` | No longer serving traffic. Stopped to avoid consuming free tier compute hours. Not terminated in case it is needed for reference |
| Reduced ASG desired capacity back to 1 after stress test | The extra instances launched during the stress test are only needed for testing. Scaling back down avoids unnecessary free tier hour consumption |

---

## 🔹 What's Next

**Lab 6** closes the loop on the one remaining manual step: deploying code. Every time the Flask application needs an update, someone currently has to SSH into the server, pull the latest code, and restart Flask. Lab 6 replaces that process with a CI/CD pipeline using AWS CodePipeline and CodeDeploy. A push to the GitHub repository triggers an automatic deployment to every instance in the ASG. Combined with everything built across Labs 1 through 5, this makes the infrastructure fully automated from scaling to deployment.

---

*Documentation by Jayvee Dela Rosa | Based on the AWS Network Challenge 2 by [Raphael Jambalos](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)*


