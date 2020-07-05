---
title: "Serverless Postgres"
date: 2020-07-04T20:52:00-05:00
draft: true
---

# Amazon Aurora Serverless PostgreSQL

## Guide

[Background](#background)  
[AWS Setup ](#aws-setup)

- [Create Accounts](#create-accounts)
- [Create an Amazon Aurora Database Cluster](#create-an-amazon-aurora-database-cluster)
- [Create an EC2 Instance](#create-an-ec2-instance)

[Local Machine Setup](#local-machine-setup)  
[Connect to the Database](#connect-to-the-database)

- [Log into EC2](#log-into-ec2)
- [Connect with EC2](#connect-with-ec2)
- [Locally Connect](#locally-connect)

[Aurora Serverless Considerations](#aurora-serverless-considerations)  
[Resources](#resources)

---

## Background

Amazon Aurora Serverless is an on-demand, auto-scaling configuration for Amazon Aurora (MySQL-compatible and PostgreSQL-compatible editions), where the database will automatically start up, shut down, and scale capacity up or down based on your application's needs.

The guide provides comprehensive instructions on setting up a serverless geospatial PostgreSQL database in AWS and connecting to it with several GIS clients.

---

## AWS Setup

### Create Accounts

1. [Sign up for AWS](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_SettingUp_Aurora.html#CHAP_SettingUp_Aurora.SignUp).

   If you don't have one already, sign up for an AWS account. This account is considered to be the _root_ user.

1. [Create an IAM User](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_SettingUp_Aurora.html#CHAP_SettingUp_Aurora.IAM).

   > As a best practice, do not use the AWS account root user for any task where it's not required. Instead, create a new IAM user for each person that requires administrator access. Then make those users administrators by placing the users into an "Administrators" group to which you attach the AdministratorAccess managed policy. All future interaction should be through the AWS account's users and their own keys instead of the root user.

   When complete, log out of AWS and log back in with the new adminstrative user created.

### Create an Amazon Aurora Database Cluster

You can only create an Amazon Aurora DB cluster in a virtual private cloud (VPC) based on the Amazon VPC service, in an AWS Region that has at least two Availability Zones. The DB subnet group that you choose for the DB cluster must cover at least two Availability Zones. This configuration ensures that your DB cluster always has at least one DB instance available for failover, in the unlikely event of an Availability Zone failure.

Configuring a new VPC and DB subnets is beyond the goal of this tutorial. For efficiency, they will be automatically created during the process of creating the database cluster.

Please note Aurora is not part of the AWS [free tier](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc). You pay for database storage, plus the database capacity and I/O your database consumes while it is active. Billing is in Aurora Capacity Units (ACU) currently set at \$0.06 per ACU hour.

These instructions were adapted from [Creating a DB Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CreateInstance.html) and [Creating an Aurora Serverless Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.create.html).

1.  Sign in (with IAM user) to the AWS Management Console and open the Amazon RDS console at https://console.aws.amazon.com/rds/.

1.  In the upper-right corner of the AWS Management Console, choose the AWS Region in which you want to create the DB cluster.

1.  In the navigation pane, choose **Databases**.

1.  Choose **Create database**.

1.  In **Choose a database creation method**, select **Standard Create**.

    ![standard-create](standard-create.jpg)

1.  In **Engine options**:

    - For **Engine type**, choose **Amazon Aurora**
    - For **Edition**, choose **Amazon Aurora with PostgreSQL compatibility**
    - In **Version**, choose **Aurora PostgreSQL (Compatible with PostgreSQL 10.7)**

    ![engine-options](engine-options.jpg)

1.  In **Database Features**, choose **Serverless**.

    ![database-features](database-features.jpg)

1.  In the **Settings** section:

    - Enter a DB cluster identifier of your choice.
    - In the **Credential Settings** dropdown, autogenerate a postgres user password or explicitly set one.

    ![settngs](settings.jpg)

1.  In the **Capacity settings** section:

    - Set **Minimum Aurora capacity unit** and **Maximum Aurora capacity unit**. Recommendation is to start with minimum of **2** and maximum **4**. These values can be changed after creation of the database cluster.

    - In the **Additional scaling configuration** dropdown, check the box **Pause compute capacity after consecutive minutes of inactivity**. Change the default setting of 5 minutes if desired.

    ![capacity-settings](capacity-settings.jpg)

1.  In the **Connectivity** section:

    - Select **Create new VPC** under **Virtual Private Cloud (VPC)**.
    - Open the **Additional connectitity configuration** dropdown.
    - For **Subnet Group**, choose **Create new DB Subnet Group**.
    - For **VPC security group**, choose **Create new** and enter a name of your choice.

    ![connectivity](connectivity.jpg)

1.  Open the **Additional configurations** dropdown.

    - For **Initial database name**, enter a database name of your choice.

    ![additional-config](additional-config.jpg)

1.  Lastly, click **Create Database** at the bottom of the page.

### Create an EC2 Instance

Before creating a new EC2 instance it is necessary to create a Key Pair and and security group.

#### Create a Key Pair for the EC2 Instance

AWS uses public-key cryptography to secure the login information for your instance. A Linux instance has no password; you use a key pair to log in to your instance securely.

1. Follow the instructions to [Create a Key Pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html#create-a-key-pair). We'll be using OpenSSH to log in so make sure to create a **pem** file. Ignore the last step on changing the **pem** file permissions.

1. Remember the name you provide for the key pair, which will be needed when launching an EC2 instance.

1. Save the **pem** file anywhere on your local machine and further instructions will be given later on how to use it to SSH into the EC2 instance.

#### Create a Security Group

Security groups act as a firewall for associated instances, controlling both inbound and outbound traffic at the instance level. You must add rules to a security group that enable you to connect to your EC2 instance from your IP address using SSH.

1. Open the VPC Console at https://console.aws.amazon.com/vpc.
1. Click **Security Groups** in the left panel.
1. Click the **Create Security Group** button at the top left:
   ![create-security-group](create-security-group.jpg)
1. Provide a name, description and select the VPC to group belongs to, then click **Create**:
   ![sg-form](sg-form.jpg)
   Jot down the name of the security group as you will use it later.
1. Click the **\*Security Group ID** link for the group that was created:
   ![sg-created](sg-created.jpg)
1. Click **Inbound Rules** for the newly created security group, then click **Edit Rules**:
   ![sg-inbound-rules](sg-inbound-rules.jpg)
1. Filter and select _SSH_ in the **Type** dropdown. Select _My IP_ in the **Source** dropdown, which will autopopulate your IP address.
   ![sg-rules](sg-rules.jpg)
1. Click **Save Rules**.

### Launch an EC2 Instance

1.  Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.

1.  From the console dashboard, choose Launch Instance.

1.  Amazon Machine Images (AMIs) that serve as templates for your instance are list. Select **Amazon Linux 2 AMI (HVM), SSD Volume Type**, which is in the free tier.

    ![ec2-type](ec2-type.jpg)

1.  On the Choose an Instance Type page, yselect the _t2.micro_ type, which is selected by default. Notice that this instance type is also eligible for the free tier.

1.  Click the **Next: Configure Instance Details** button.

1.  Ensure the **Network** value in the dropdown is the same VPC as the Aurora database cluster created earlier. If they are in different VPCs, you will not be able to connect the EC2 instance to the database.

    Make other configurations as needed, but the defaults should suffice.

1.  Click the **Review and Launch** button.

1.  On the Review Instance Launch page, under Security Groups, you'll see that the wizard created and selected a security group for you. You will not use this group. Instead, you wil select the security group created earlier:

    - Choose Edit security groups.
    - On the Configure Security Group page, ensure that Select an existing security group is selected.
    - Select the security group that was previously created, and then click the **Review and Launch** button.


        ![ec2-sg](ec2-sg.jpg)

1. On the Review Instance Launch page, click the **Launch** button. When prompted for a key pair, select Choose an existing key pair, then select the key pair that you created earlier.

   ![ec-keypair](ec2-keypair.jpg)

1. Patiently wait until the instance is spun up.

## Local Machine Setup

You'll now set up your machine to SSH into the EC2 instance, which will allow you to connect to the Aurora Serverless PostgreSQL database. Linux will be used primarily for its built-in support for OpenSSH. If you are running Windows 10, you can install Windows Subsystem for Linux (WSL).

1. If running Windows 10, install [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) (WSL) if using Windows 10. See [WSL Installation Guide for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10) for instructions.

1. Install [AWS CLI for Linux](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html). The `unzip` utility will need to be installed for WSL, if not already. In addition, use the curl `-k` option to download the AWS CLI installation file. This allows insecure server connections when using SSL. Otherwise, use a browser to download.

   Open a (WSL) terminal and execute the following commands:

   ```bash
   # Install unzip
   sudo apt install unzip
   ```

   ```bash
   # Download cli
   curl -k "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   ```

   ```bash
   # Unzip and install the cli
   unzip awscliv2.zip
   sudo ./aws/install
   ```

   ```bash
   # Confirm install
   aws --version
   ```

1. An access key is required in order to sign requests that you make using Amazon CLI that will be used. Go to [Quickly Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration) and scroll down to the section named **To create access keys for an IAM user**. Follow the instructions to generate keys for the _Administrator_ IAM user created earlier. You will use the public and private key values in the next step.

1. Now, configure the AWS CLI with the previously generated keys, your default region, and output format of choice:

   ```bash
   aws configure
   ```

   ```bash
   # Enter configuration values when prompted
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   Default region name [None]: us-east-2
   Default output format [None]: json
   ```

   | Option         | Values                                                                                                |
   | -------------- | ----------------------------------------------------------------------------------------------------- |
   | Region name    | See [regional endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html#regional-endpoints) |
   | Output formats | _json, yaml, text, table_                                                                             |

1. Later on, example database connections will be made with pgAdmin, qQGIS and ArcGIS Pro. At least one of these clients will need to be locally installed:

   - [pgAdmin](https://www.pgadmin.org/download/)
   - [qGIS](https://qgis.org/en/site/forusers/download.html)
   - ArcGIS Pro ([free trial](https://www.esri.com/en-us/arcgis/trial?rmedium=esri_com_redirects01&rsource=https://links.esri.com/pro/trial))

---

## Connect to the Database

You can't give an Aurora Serverless DB cluster a public IP address. You can access an Aurora Serverless DB cluster only from within a virtual private cloud (VPC) based on the Amazon VPC service. Since the EC2 instance created earlier is in the same VPC as the DB cluster, it has access to connect.

In this section you will SSH from your local machine into the EC2 instance. You can then use pSQL to connect to the PostgreSQL database. You'll then set up an SSH tunnel so that you can connect with other local clients like pgAdmin, qGIS and ArcGIS Pro.

### Log into EC2

1. Open a terminal.

1. Earlier you created a **pem** Key Pair file during the EC2 setup. Locate this file and move (avoid copying) it to somewhere accessible. A good place is the .ssh directory under your home.

   ```bash
    mv aws-ec2.pem ~/.ssh
   ```

1. Use the following command to set the permissions of your private key file so that only you can read it.

   ```bash
   cd ~/.ssh
   chmod 400 aws-ec2.pem
   ```

1. Now connect to the EC2 instance via SSH. You may need to look up the public DNS for the EC2 instance in the Amazon Console. Replace the endpoint below with your own. Example: [_ec2-3-15-21-102.us-east-2.compute.amazonaws.com_]()

   ```bash
   cd ~
   sudo ssh ec2-user@ec2-3-15-21-102.us-east-2.compute.amazonaws.com -i ~/.ssh/aws-ec2.pem
   ```

   If prompted, enter your sudo password and you will be logged into your EC2 instance:

   ```bash
   sudo password:

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
       ___|\___|___|

       https://aws.amazon.com/amazon-linux-2/
       [ec2-user@ip-172-30-0-41 ~]$
   ```

   Remain connected via SSH for the remaining steps.

### Connect with EC2

1. Next, install psql on the EC2 instance:

   ```bash
   sudo yum install postgresql
   ```

1. Now, connect to the datase. You'll need to know the DB cluster endpoint for this step, which is available in the Amazon Console. This will be the value for the _-h_ (host) option. The _-d_ option is the name of the database. Execute this psql command which will prompt you to enter the postgres database password.

   ```bash
   # replace the -h and -d values appropriately
   psql -h database-1.cluster-clih9gfnqsno.us-east-2.rds.amazonaws.com -d geo -U postgres
   ```

   Alternatively use an environment variable to set the password up front.

   ```bash
   # Set password to avoid being prompted
   PGPASSWORD=************** psql -h database-1.cluster-clih9gfnqsno.us-east-2.rds.amazonaws.com -d geo -U postgres
   ```

   Once connected, a postgres prompt will appear.

   ```bash
   geo=>
   ```

1. Stay connected to the database and run the following SQL to enable PostGIS:

   Verify the version:

   ```bash
   geo=> SELECT PostGIS_full_version();

   POSTGIS="2.4.4" PGSQL="100" GEOS="3.6.2-CAPI-1.10.2 4d2925d6" PROJ="Rel. 4.9.3, 15 August 2016" GDAL="GDAL 2.1.4, released 2017/06/23" LIBXML="2.9.3" LIBJSON="0.12.99" LIBPROTOBUF="1.3.0" (core procs from "2.4.4" need upgrade) RASTER (raster procs from "2.4.4" need upgrade)
   ```

   Install PostGIS:

   ```bash
   geo=> CREATE EXTENSION postgis;
   ```

   Other PostGIS extensions that can be enabled in Aurora:

   ```bash
   geo=> CREATE EXTENSION postgis_tiger_geocoder;
   geo=> CREATE EXTENSION postgis_topology;
   geo=> CREATE EXTENSION postgis_fdw;
   ```

   ```bash
   # List all supported extensions
   geo=> SHOW rds.extensions;
   ```

1. Exit psql

   ```bash
   geo=> \q
   ```

### Locally Connect

Since the datase is in a VPC that doesn't allow public access, an SSH tunnel will be used in order to make local connections to the database. You will set up a port on your machine to listen for connections which will be forwarded to the Aurora serverless PostgreSQL database through the EC2 instance.

1. Disconnect from EC2, if needed. The tunnel will be created locally.

1. At the terminal, use the _ssh_ command to create the tunnel. Your machine will listen for database connections on port 5433 and forward them to Aurora PostgreSQL database port 5432 using a secured connection to your EC2 instance.

   ```bash
   ## Create SSH tunnel
   sudo ssh -N -L 5433:database-1.cluster-clih9gfnqsno.us-east-2.rds.amazonaws.com:5432 ec2-user@ec2-3-15-21-102.us-east-2.compute.amazonaws.com -i .ssh/aws-ec2.pem
   ```

   Make sure to replace the values for database cluster endpoint, EC2 host, and \*.pem file.

   The tunnel must be running to make connections.

1. Install psql locally.

   ```bash
   sudo apt-get install postgresql-client-10
   ```

1. Connect to the database.

   ```bash
   psql -h localhost -p 5433 -U postgres -d geo
   password for user postgres: ************
   ```

   ```bash
   geo=>
   ```

### pgAdmin

1. When creating a new server connection, jump to the **SSH Tunnel** tab. Turn on SSH tunneling and provide your EC2 endpoint and user. Switch authentication from **Password** to **Identity file** and point to the location of your \*.pem file.

   ![pg-tunnel](pg-tunnel.jpg)

1. Switch over to the **Connection** tab and provide the database cluster endpoint for the host, port 5432 and the database user (usually postgres) and password.

   ![pg-conn](pg-conn.jpg)

1. Upon saving the connection you should now see the database.

   ![pg-dbs](pg-dbs.jpg)

### qGIS

When creating a connection in qGIS, specify the following:

**Host**: localhost  
 **Port**: 5433 (local port set up in SSH tunnel)  
 **Database**: your DB name  
 **SSL mode**: allow

![qGIS connection](qgis-conn.jpg)

Test the connection and be on your way.

### ArcGIS

When creating a connection in ArcGIS Pro, specify the following:

**Database Platform**: PostgreSQL  
 **Instance**: 127.0.0.1,5433  
 **Authentication Type**: Database authentication  
**User Name**: postges  
**Password**: _user password_  
**Database**: your database name

![ArcGIS Pro connection](arcgis-pro-conn.jpg)

---

## Aurora Serverless Considerations

### Advantages

**Simpler**  
Aurora Serverless removes much of the complexity of managing DB instances and capacity.

**Scalable**  
Aurora Serverless seamlessly scales compute and memory capacity as needed, with no disruption to client connections.

**Cost-effective**  
When you use Aurora Serverless , you pay for only the database resources that you consume, on a per-second basis.

**Highly available storage**  
Aurora Serverless uses the same fault-tolerant, distributed storage system with six-way replication as Aurora to protect against data loss.

Source: [Advantages of Aurora Serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.advantages).

### Use Cases

**Infrequently used applications**  
You have an application that is only used for a few minutes several times per day or week, such as a low-volume blog site. With Aurora Serverless , you pay for only the database resources that you consume on a per-second basis.

**New applications**  
You are deploying a new application and are unsure about which instance size you need. With Aurora Serverless , you can create a database endpoint and have the database autoscale to the capacity requirements of your application.

**Variable workloads**  
You're running a lightly used application, with peaks of 30 minutes to several hours a few times each day, or several times per year. Examples are applications for human resources, budgeting, and operational reporting applications. With Aurora Serverless , you no longer need to provision to either peak or average capacity.

**Unpredictable workloads**  
You're running workloads where there is database usage throughout the day, but also peaks of activity that are hard to predict. An example is a traffic site that sees a surge of activity when it starts raining. With Aurora Serverless , your database autoscales capacity to meet the needs of the application's peak load and scales back down when the surge of activity is over.

**Development and test databases**  
Your developers use databases during work hours but don't need them on nights or weekends. With Aurora Serverless , your database automatically shuts down when it's not in use.

**Multi-tenant applications**  
With Aurora Serverless , you don't have to individually manage database capacity for each application in your fleet. Aurora Serverless manages individual database capacity for you.

Source: [Use Cases for Aurora Serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.use-cases).

### Limitations

- Aurora Serverless is only available for the following:
  - Aurora with MySQL version 5.6 compatibility
  - Aurora with PostgreSQL version 10.7 compatibility
- The port number for connections must be:
  - 3306 for Aurora MySQL
  - 5432 for Aurora PostgreSQL
- You can't give an Aurora Serverless DB cluster a public IP address. You can access an Aurora Serverless DB cluster only from within a virtual private cloud (VPC) based on the Amazon VPC service.
- Each Aurora Serverless DB cluster requires two AWS PrivateLink endpoints. If you reach the limit for AWS PrivateLink endpoints within your VPC, you can't create any more Aurora Serverless clusters in that VPC. For information about checking and changing the limits on endpoints within a VPC, see Amazon VPC Limits.
- A DB subnet group used by Aurora Serverless can’t have more than one subnet in the same Availability Zone.
- Changes to a subnet group used by an Aurora Serverless DB cluster are not applied to the cluster.
- A connection to an Aurora Serverless DB cluster is closed automatically if it stays open for longer than one day.
- Aurora Serverless doesn't support the following features:
  - Loading data from an Amazon S3 bucket
  - Saving data to an Amazon S3 bucket
  - Invoking an AWS Lambda function with an Aurora MySQL native function
  - Aurora Replicas
  - Backtrack
  - Multi-master clusters
  - Database cloning
  - IAM database authentication
  - Restoring a snapshot from a MySQL DB instance
  - Amazon RDS Performance Insights

Note: You can access an Aurora Serverless DB cluster from AWS Lambda.

Source: [Limitations of Aurora Serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html#aurora-serverless.limitations).

---

## Additonal Resources

### AWS

[Using Amazon Aurora Serverless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless.html)  
[AWS CLI Command Line Reference](https://docs.aws.amazon.com/cli/latest/reference)

### SSH Tunneling

[How to Create SSH Tunnels](https://www.tunnelsup.com/how-to-create-ssh-tunnels/)  
[Connect to RDS Using an SSH Tunnel](https://medium.com/@michalisantoniou6/connect-to-an-aws-rds-using-an-ssh-tunnel-22f3bd597924)  
[Secure PostgreSQL TCP/IP Connections with SSH Tunnels](https://www.postgresql.org/docs/8.2/ssh-tunnels.html)  
[Access Your Database Remotely Through an SSH Tunnel](https://support.cloud.engineyard.com/hc/en-us/articles/205408088-Access-Your-Database-Remotely-Through-an-SSH-Tunnel)

### PostgreSQL and PostGIS

[psql Reference](https://www.postgresql.org/docs/10/app-psql.html)
