# Lab 3: Use Managed Services to Make Your Life Easier

**Status:** ✅ Complete

**Date Completed:** May 3, 2026

**Reference:** [AWS Network Challenge 2 by Raphael Jambalos](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)

---

## 🔹 Overview

Lab 3 is where the architecture stops feeling like a science project and starts feeling like a real cloud deployment. In Lab 2, I was manually managing four EC2 instances, including a MongoDB server and a PostgreSQL server that I had to configure, secure, and keep running myself. Lab 3 replaces those two database servers with AWS managed services, and replaces the Nginx Proxy Server with an Application Load Balancer. By the end of this lab, I went from managing four EC2 instances down to one.

The core idea is simple: instead of running databases on servers I have to babysit, I hand that responsibility to AWS. Amazon RDS manages PostgreSQL. Amazon DocumentDB manages MongoDB. The Application Load Balancer manages traffic routing. My Flask app does not care about any of this. It just reads from environment variables and connects to whatever hostname I point it at.

![Lab 3 Architecture](images/lab3/lab3-architecture-diagram.png)

*Source: [Raphael Jambalos — AWS Network Challenge 2](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)*

---

## 🔹 Goal

Replace the manually managed database EC2 servers and Nginx Proxy Server with AWS managed services:

- Replace the PostgreSQL EC2 server with **Amazon RDS**
- Replace the MongoDB EC2 server with **Amazon DocumentDB**
- Replace the Nginx Proxy Server EC2 with an **AWS Application Load Balancer**

The Flask application code does not change. Only what the environment variables point to changes.

---

## 🔹 What I Built

**AWS Resources Created:**

- 2 new Security Groups (`flask-app-rds-sg`, `flask-app-docdb-sg`)
- 1 DB Subnet Group (`flask-app-db-subnet-group`)
- 1 second private subnet (`flask-app-private-subnet-2`, `10.0.3.0/24`, ap-southeast-1b)
- 1 second public subnet (`flask-app-public-subnet-2`, `10.0.4.0/24`, ap-southeast-1b)
- 1 Amazon RDS PostgreSQL instance (`flask-photo-app-postgres`, db.t3.micro)
- 1 Amazon DocumentDB cluster (`flask-photo-app-docdb`, db.t3.medium, engine 8.0.0)
- 1 DocumentDB Cluster Parameter Group (`flask-app-docdb-params`) with TLS disabled
- 1 ALB Security Group (`flask-app-alb-sg`)
- 1 Target Group (`flask-app-target-group`)
- 1 Application Load Balancer (`flask-app-alb`)
- 1 temporary NAT Gateway (created and deleted during troubleshooting)

**EC2 Instances Stopped:**

- `flask-app-mongodb-server`
- `flask-app-postgresql-server`
- `flask-app-proxy-server`

---

## 🔹 Code Integration

The Flask application (`main.py`) was not modified. The only changes were to environment variables and one connection file.

**Environment variable changes:**

| Variable | Lab 2 Value | Lab 3 Value |
|---|---|---|
| `MONGODB_DB_CONNECTION_URI` | `mongodb://10.0.2.166:27017/` | `mongodb://flaskadmin:PASSWORD@YOUR_DOCDB_ENDPOINT:27017/` |
| `POSTGRESQL_DB_HOST` | `10.0.2.239` | `flask-photo-app-postgres.czg6yog6algm.ap-southeast-1.rds.amazonaws.com` |
| `POSTGRESQL_DB_USERNAME` | `flask_photo_app_admin` | `flask_app_admin` |
| All others | Unchanged | Unchanged |

**`db/mongodb/mongodb_connection.py` change:**

Both `MongoClient(uri)` calls were updated to:

```python
client = MongoClient(uri, tls=False, serverSelectionTimeoutMS=30000)
```

This was required to explicitly disable TLS at the driver level and increase the connection timeout. The reason for this change is explained in the DocumentDB troubleshooting section below.

---

## 🔹 My Experience

### Creating Security Groups and DB Subnet Group

The first thing I built was two new security groups, one for RDS and one for DocumentDB. Both follow the same principle from Lab 2: the source is not an IP address but another security group. `flask-app-rds-sg` allows port 5432 only from `flask-app-server-sg`. `flask-app-docdb-sg` allows port 27017 only from `flask-app-server-sg`. This means only the App Server can ever reach either database, regardless of what IP it has.

![New Security Groups](images/lab3/lab3-01-new-securitygroups.png)

*Both new security groups created, one for RDS and one for DocumentDB*

AWS requires a DB Subnet Group before creating either RDS or DocumentDB. The subnet group tells AWS which subnets the managed services are allowed to deploy into, and AWS requires at least two subnets in two different Availability Zones. This meant I had to create a second private subnet in ap-southeast-1b before I could proceed. That subnet has no servers in it. It exists purely to satisfy this requirement.

![DB Subnet Group](images/lab3/lab3-02-db-subnet-group.png)

*DB Subnet Group created with both private subnets across two Availability Zones*

---

### Creating Amazon RDS

Creating the RDS instance was straightforward. I selected PostgreSQL as the engine, chose the Dev/Test template, selected Single-AZ deployment, and placed it inside `flask-photo-app-vpc` using the `flask-app-db-subnet-group`. The most important fields were Public access set to No, and the Initial database name set to `ecv_file_upload_app_psql`. Skipping that database name field is a common mistake that causes the app to connect successfully but immediately fail because no database exists inside the instance yet.

Two things came up during RDS creation that deviated from the original setup values in Sir Raphael's repository. First, RDS has a 16-character limit on master usernames, so flask_photo_app_admin from the original repository was too long. I used flask_app_admin instead. Second, the original password was rejected because RDS does not allow the @ character in passwords, as it is a reserved character in database connection strings. I replaced the @ with ! to satisfy the constraint. Both deviations required updating the corresponding environment variables on the App Server to match.

After about eight minutes, the instance reached Available status and I copied the endpoint:

![RDS Available](images/lab3/lab3-03-rds-available.png)

*RDS instance showing Available status with the endpoint ready to copy*

---

### Creating Amazon DocumentDB

DocumentDB was the most complex part of this lab, and where I spent the majority of my time. The creation itself was mostly straightforward, but there was one critical configuration step that caused hours of troubleshooting later: TLS.

By default, DocumentDB requires TLS-encrypted connections. The pymongo connection string in the Flask app does not include TLS parameters, which means a plain connection attempt will fail with a connection reset error. The correct approach is to disable TLS on the cluster using a custom parameter group.

The AWS Console no longer shows a TLS toggle during cluster creation. Instead, I had to create a separate Cluster Parameter Group first, find the `tls` parameter inside it, set it to `disabled`, and then select that parameter group during cluster creation. I created `flask-app-docdb-params` with the `tls` parameter set to `disabled` before creating the cluster itself.

![DocumentDB Available](images/lab3/lab3-04-docdb-available.png)

*DocumentDB cluster showing Available status with the endpoint visible*

DocumentDB also required alphanumeric-only usernames, so `flask_app_admin` with an underscore was rejected. I used `flaskadmin` as the DocumentDB master username.

---

### Initializing RDS and Restarting Flask

With both managed databases available, I SSHed into the App Server through the Proxy Server and updated all eight environment variables to point to the new RDS and DocumentDB endpoints. Then I ran `db/postgresql/init_db.py` to create the `products` and `stock_movements` tables on the fresh RDS instance.

The first attempt at running init_db.py failed with a hostname resolution error. The cause was extra quotation marks in the `POSTGRESQL_DB_HOST` export command, which made the hostname `""flask-photo-app-postgres...""`  with literal double quotes included. Re-exporting the variable with only single quotes fixed it immediately. The second run completed with no output, which means success.

![init_db.py Success](images/lab3/lab3-05-initdb-rds-success.png)

*init_db.py completing with no errors, confirming the PostgreSQL tables were created on RDS*

After restarting Flask inside the tmux session with the new environment variables, Flask started cleanly with no database connection errors in the output.

---

### Creating the Application Load Balancer

The ALB requires subnets in at least two Availability Zones, just like the DB Subnet Group. I created a second public subnet (`flask-app-public-subnet-2`, `10.0.4.0/24`, ap-southeast-1b) and associated it with the public route table so it would have internet access through the Internet Gateway.

I created a Target Group pointing to the App Server on port 5000, with a health check path of `/`. The ALB periodically sends a GET request to that path. Flask's root route returns a 200 OK response, which tells the ALB the server is alive and healthy.

![Target Group Created](images/lab3/lab3-06-targetgroup-created.png)

*Target group created with the App Server registered as a target on port 5000*

I also updated the App Server's security group inbound rule for port 5000 to allow traffic from `flask-app-alb-sg` instead of `flask-app-proxy-sg`. This is the step that hands traffic routing responsibility from Nginx to the ALB.

The ALB itself was configured as internet-facing, listening on port 80, forwarding to the target group. After about three minutes it reached Active status.

![ALB Active](images/lab3/lab3-07-alb-active.png)

*Application Load Balancer showing Active status with the DNS name visible*

---

### The DocumentDB Connection Problem

This was the single most frustrating part of Lab 3, and the longest troubleshooting session I have had in this entire project. After everything was set up and Flask was running, visiting `/images` returned an Internal Server Error. The root route and the upload form page both loaded fine, but anything that queried DocumentDB failed.

The error in the Flask logs was `ServerSelectionTimeoutError` followed by `Connection reset by peer`. The network was not the issue. A Python socket test confirmed that port 27017 on the DocumentDB endpoint was reachable. Security groups were correct. The connection string was correct. But pymongo kept timing out or getting its connection reset.

I tried multiple approaches: different URI parameter combinations, a pymongo version downgrade from 4.10.1 to 4.6.3 (which required creating a temporary NAT Gateway to give the private App Server internet access), and modifying `mongodb_connection.py` to pass connection parameters directly to the MongoClient constructor instead of relying on the URI string alone.

None of these fixed it completely on their own. The real cause turned out to be a single missed step: the DocumentDB cluster had never been rebooted after the parameter group change. AWS applies parameter group changes only after a reboot. The cluster was running with TLS still enabled even though the parameter group showed TLS as disabled. Every connection attempt was being rejected at the TLS handshake level.

Rebooting the cluster instance took about five minutes. After it came back up, everything worked immediately.

The pymongo downgrade and the `mongodb_connection.py` code change both stayed in place as complementary fixes. The downgrade ensures compatibility with DocumentDB 8.0, and the explicit `tls=False` parameter in the MongoClient constructor adds a belt-and-suspenders layer that prevents TLS negotiation even if the parameter group setting is ever accidentally changed.

![Target Group Healthy](images/lab3/lab3-10-targetgroup-healthy.png)

*Target group showing healthy status after all fixes were applied*

---

### Stopping the Old Servers

With the ALB, RDS, and DocumentDB all working, I stopped the three EC2 instances that were no longer needed: `flask-app-mongodb-server`, `flask-app-postgresql-server`, and `flask-app-proxy-server`. I chose Stop rather than Terminate to keep a rollback option available until after the May 9 presentation.

![Old Servers Stopped](images/lab3/lab3-08-oldservers-stopped.png)

*Old MongoDB, PostgreSQL, and Proxy Server EC2 instances in Stopped state*

---

## 🔹 Final Verification

After all fixes were applied and the cluster was rebooted, all three routes worked correctly through the ALB DNS name.

![Hello World via ALB](images/lab3/lab3-09-helloworld-alb.png)

*Flask app returning Hello World via the ALB DNS name, confirming end-to-end routing works*

![Upload Form via ALB](images/lab3/lab3-11-upload-form-alb.png)

*Upload form working via ALB, with files saving to disk and metadata reaching DocumentDB*

![Images Page via ALB](images/lab3/lab3-12-images-page-alb.png)

*Images page loading correctly via ALB, retrieving metadata from DocumentDB*

---

## 🔹 Errors and Fixes Summary

| Error | Cause | Fix |
|---|---|---|
| RDS master username rejected | `flask_photo_app_admin` exceeds RDS 16-character limit | Used `flask_app_admin` instead and updated the environment variable to match |
| RDS password rejected | `@` character is not allowed in RDS passwords | Replacing `@` with `!` |
| `init_db.py` failed with hostname error | Extra double quotes inside the `POSTGRESQL_DB_HOST` export command | Re-exported the variable with only single quotes wrapping the hostname |
| DocumentDB username rejected | Underscore not allowed, DocumentDB requires alphanumeric usernames only | Used `flaskadmin` instead of `flask_app_admin` |
| `/images` Internal Server Error after setup | DocumentDB cluster was never rebooted after parameter group change, so TLS was still enabled despite the parameter group showing disabled | Rebooted the DocumentDB cluster instance from the AWS Console |
| pymongo `ServerSelectionTimeoutError` persisting | pymongo 4.10.1 had compatibility issues with DocumentDB 8.0 | Downgraded pymongo to 4.6.3 using a temporary NAT Gateway for internet access |
| `Connection reset by peer` error | TLS still active at handshake level before reboot | Modified `mongodb_connection.py` to pass `tls=False` directly to MongoClient constructor |
| Environment variables lost between sessions | `export` commands only persist for the current terminal session | Re-exported all variables inside the tmux session where Flask runs |
| SSH connection timed out to App Server | App Server security group SSH rule was set to a specific home IP | Temporarily changed SSH source to `0.0.0.0/0`, safe because the App Server has no public IP |
| Flask still connecting to old MongoDB EC2 | Environment variables set outside tmux were not visible inside tmux | Learned to always export variables and start Flask inside the same tmux session |

---

## 🔹 Key Learnings

**1. Managed services eliminate operational work, not complexity.**

Switching from EC2 databases to RDS and DocumentDB removed the need to manage mongod.conf, pg_hba.conf, listen_addresses, and manual replication. But it introduced new concepts like DB Subnet Groups, parameter groups, and cluster reboots. The operational burden goes down significantly. The conceptual surface area does not disappear. You trade one kind of knowledge for another.

**2. Parameter group changes require a reboot to take effect.**

This was the lesson that cost me the most time in Lab 3. I correctly configured the DocumentDB parameter group to disable TLS. I confirmed the parameter group was applied to the cluster. But I never rebooted the cluster instance, so the change never actually took effect. The cluster was still running with TLS enabled the entire time. AWS shows the parameter group as applied, but it will not activate until the instance is rebooted. This is documented behavior that I simply did not know about. I know it now.

**3. "Connection reset by peer" means the server is actively rejecting you, not ignoring you.**

There is an important difference between a timeout and a connection reset. A timeout means the server is unreachable or not responding. A connection reset means the server received the connection, decided it did not like something about it, and actively closed it. In this case, DocumentDB was closing the connection because it was expecting TLS and the client was not negotiating TLS. Understanding this distinction helped narrow down the root cause much faster than blindly retrying with different connection strings.

**4. The ALB requires two public subnets across two Availability Zones, even for single-AZ deployments.**

Just like the DB Subnet Group, the Application Load Balancer cannot be created with only one subnet. AWS requires at least two subnets in two different AZs for both managed database services and load balancers. This is an architectural requirement tied to high availability design, even when you are not actually using high availability. I created `flask-app-public-subnet-2` in ap-southeast-1b purely to satisfy this requirement. It has no resources in it.

**5. Never change a configuration and assume it is active.**

The reboot lesson generalizes into something bigger. When you change a configuration in any system, whether it is a database parameter group, a security group rule, or an environment variable, always verify that the change has actually been applied before concluding it did not work. I spent hours debugging the wrong layer because I assumed the TLS parameter was active when it was only pending. The lesson is to confirm activation, not just confirmation of the setting itself.

---

## 🔹 Cleanup Performed

| Action | Reason |
|---|---|
| Stopped `flask-app-mongodb-server` EC2 | Replaced by DocumentDB. Kept in Stopped state as rollback option until after May 9 presentation |
| Stopped `flask-app-postgresql-server` EC2 | Replaced by RDS. Same reason |
| Stopped `flask-app-proxy-server` EC2 | Replaced by ALB. Same reason |
| Deleted temporary NAT Gateway | Created only to allow pymongo downgrade on the private App Server. Deleted immediately after to stop charges |
| Released temporary Elastic IP | Associated with the temporary NAT Gateway. Released after NAT Gateway was deleted |
| Removed `directConnection=True` from MongoClient | Simplification after confirming the reboot fixed the root cause. Kept `tls=False` and `serverSelectionTimeoutMS=30000` |

---

## 🔹 What's Next

**Lab 4** addresses a problem that only becomes visible once the architecture works: the uploaded images are stored directly on the App Server EC2 disk. If a second App Server is ever added, images uploaded on the first server will not be visible when a request is served by the second. The file storage is tied to one machine.

Lab 4 fixes this by introducing **Amazon EFS**, a shared file system that can be mounted on multiple EC2 instances simultaneously. When a user uploads an image, it goes to the EFS volume instead of the local disk. Both app servers see the same files immediately. This is a prerequisite for Lab 5, where auto-scaling will add and remove App Server instances automatically.

---

*Documentation by Jayvee Dela Rosa | Based on the AWS Network Challenge 2 by [Raphael Jambalos](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)*

