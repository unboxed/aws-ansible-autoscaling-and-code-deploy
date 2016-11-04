# Overview

This repository demonstrates one way of building an autoscaling app on AWS using
Ansible, CloudFormation, and CodeDeploy.

A presentation explaining how this fits together is available in the docs/
directory.

There's also a simplistic matching application available at
https://github.com/unboxed/aws-ansible-autoscaling-and-code-deploy-app
which shows how the CodeDeploy steps work.

# Important Note

Make sure you read the LICENSE.txt file!

This is a demo environment. Your mileage may vary! Make sure you understand the
cost and security implementations of running this against your AWS account.

You should probably create an AWS sub-account to ensure you don't break
anything important in your real environment. http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html
has more information.

# Assumptions

To try and keep this as simple as possible, I've made various assumptions about
your environment.

1. We only currently support the us-east-1 region (Ireland). A
  search-and-replace should fix that.
2. This repository only supports Ubuntu 14.04
3. You want O/S security upgrades applied everywhere automatically. (See the
  ansible-apt section of ansible/host_vars/all for details)
4. SSL only site. This does mean you need to create an SSL certificate, or
  change the Nginx config to avoid redirecting non-SSL traffic
5. You want each individual to have their own account. See the "System Users"
  section of ansible/host_vars/all for more details
6. All users will use SSH key-only authentication (set
  sshd_config_password_authentication to true in ansible/host_vars/all if not)
7. You are aiming for a high-traffic site - so you want Nginx and Kernel
  parameters that are optimised towards a busy server.

# Setting up a Demo System

## AWS Prep

1. Install the most recent AWS CLI tools (```brew install awscli``` if you are
  on a mac with homebrew, or see
  http://docs.aws.amazon.com/cli/latest/userguide/installing.html for more
  details)
2. Create AWS account or sub-account.
   - Ensure you are in us-east-1
   - Create and download an ssh key - http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html
      (Note that you will not normally use this - it's only for failsafe purposes if the
      Ansible build scripts fail.)
   - Get AWS Access Key ID and Access Secret Keys - see https://console.aws.amazon.com/iam/home?#security_credential
3. On your command line, set these environment variables (assuming the bash shell)

        export AWS_DEFAULT_REGION='us-east-1'
        export AWS_ACCESS_KEY_ID='AKI...'
        export AWS_SECRET_ACCESS_KEY='...'

4. Create a test SSL key and upload it. Take note of the 'ArnId' of the uploaded key:
  (You can retrieve it later with 'aws iam list-server-certificates')

        openssl genrsa -out $HOME/Documents/blog.example.key 2048
        openssl req -new -x509 -key $HOME/Documents/blog.example.key -out $HOME/Documents/blog.example.cert -days 3650 -subj /CN=blog.example
        aws iam upload-server-certificate --server-certificate-name blog.example --certificate-body file://$HOME/Documents/blog.example.cert --private-key file://$HOME/Documents/blog.example.key

## Build S3 Bucket Stack

First off, you're going to upload the Ansible config and the App to an S3
bucket. This has to happen first, since when the server instances come up, they
need to download and install this software. Without this, they won't come up
correctly.

I've included a simple cloudformation stack to create the buckets for you. To
use it:

Log into the AWS web console and make sure you are in the 'Ireland'
(us-east-1) region. Then:

1. Click on the "Cloudformation" service
2. Click "Create Stack"
3. Enter "blog-dev-s3-buckets" as the stack name (you will have multiple stacks
    eventually: one for dev, one for staging, etc) The exact name doesn't really
    matter.
4. Click "Choose File" and select the s3-buckets.json file you created. Then click Next.
5. Set System Parameters:
  - AppName: Choose 'blog' (lower-case) for this example.
  - BucketPrefix: Set this to something unique to you, so that you don't
    conflict with other users of this repo. use something like your-name or
    yourname or your-organisation-name (keep it lower-case and use hyphens, not
    underscores).
6. Click "Next", and "Next" again to go through the advanced options.
7. Click "Create"
8. Wait for the stack to build. If everything is successful, it should
    change to state 'CREATE_COMPLETE'.

## Repository Preparation

The next step is to configure your application - it's hostname, the environment
variables it needs, and similar.

1. Edit facts/dev/app.fact and set the hostname you want the server to respond to. You
  can also set any environment variables or similar in the same location.
2. Configure your personal account details by editing ```ansible/group_vars/all```
  Add an entry to the ```sysadmin_users``` array - by commenting out the 'sysadmin_users: []' line
  and adding a user in the format explained in that section.

## Upload the Ansible Configs

```shell
$ export AWS_DEFAULT_REGION='us-east-1'
$ export AWS_ACCESS_KEY_ID='AKI...'
$ export AWS_SECRET_ACCESS_KEY='...'
$ # Upload the ansible config - note that you must pass parameters that match
$ # the parameters you supplied to the S3 bucket stack.
$ scripts/push-config-to-s3 bucketprefix blog dev
Copying fact files to S3
...
SUCCESS
$
```

## Build the CloudFormation Stack

AWS Cloudformation configs are limited to around 50kb - and whitespace counts
against you. So you need to strip out the whitespace before doing the upload.
The simplest way to do this is to use jq:

1. Make sure you have the 'jq' app installed (```brew install jq``` on a mac with Homebrew)
2. Compact the CloudFormation config on the command line with the command ```jq < stack.json -cMa . > stack-compressed.json```.


Log into the AWS web console and make sure you are in the 'Ireland'
(us-east-1) region. Then:

1. Click on the "Cloudformation" service
2. Click "Create Stack"
3. Enter "blog-dev" as the stack name (you will have multiple stacks eventually:
    one for dev, one for staging, etc) The exact name doesn't really matter.
4. Click "Choose File" and select the stack-compact.json file you created. Then click Next.
5. Set System Parameters:
  - AppName: Choose 'blog' (lower-case) for this example.
  - BucketPrefix: Set this to something unique to you, so that you don't
    conflict with other users of this repo. use something like your-name or
    yourname or your-organisation-name (keep it lower-case and use hyphens, not
    underscores). This must match the value you previously set for the S3 stack.
  - DBAllocatedStorage: How many gigabytes of DB space to add to the server. Note that
    due to AWS pricing rules, you will have very poor performance if you create a DB
    of less than 100GB. You will hit your burst performance limit quickly. See
    http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html for more info.
    Obviously the higher the number, the bigger the cost.
  - DBInstanceType: Set this to whatever DB instance type you want. Note that small
    RDS instances can take a very long time to create, and don't support full encryption.
  - DBMaxConnections: How many DB connections you want. This reduces memory on your
    database, but allows you to set your DB pool size higher than the RDS defaults.
    https://wiki.postgresql.org/wiki/Number_Of_Database_Connections has more info on memory
    vs connection count tradeoffs. You probably want your connection count to be in
    the region of NumberOfRubyServers * NumberOfWorkers * NumberOfThreads - and the
    default app expects up 1 server, 4 workers, and up to 32 threads (128 connections
    per server.)
  - DBMultiAZ: If you're just testing things, you might want to run a single
    availability zone DB. Set to true if you want any reliability. See
    http://aws.amazon.com/rds/details/multi-az/ for details and cost implications,
    since multi-AZ dbs are expensive.
  - DBName: Set the name of the db - this will automatically have the environment
    appended to it. For this example - set it to 'blog', so that the db name will
    be created as 'blog_dev'. The associated user will be 'blog_dev_user'
  - DBPassword: set a long and complicated password that your DB will use. The
    app will be passed this through a DATABASE_URL environment variable. See
    http://12factor.net/config for details.
  - DBStorageEncrypted: Set to true if you want your DB to be encrypted at rest.
    Note that not all instance types (DBInstanceType param) support encryption.
    See http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html for
    info.
  - DBStorageType: Set the storage type. You probably want SSD (gpt).
  - EnvironmentName: What environment is this? Choose Dev for testing purposes. This
    is used for naming S3 buckets, and setting server instance permissions so that they
    can read from those buckets. Note that it is NOT used for setting the RAILS_ENV
    environment parameter. You should set that in the facts/*/app.fact file, in the
    app_environment section.
  - SecretKeyBase: This is critical for the security of your app. You will likely
    want to run ```bundle exec rake secret``` in a Rails app directory to generate
    this. This is passed into the application as an environment variable, so you
    can use it in secrets.yml in your Rails app.
  - ServerAMIID: Which AWS/Ubuntu AMI to run. See
    http://cloud-images.ubuntu.com/trusty/current/ You *must* choose a 'hvm-ssd'
    style kernel. To reduce the time spent applying security upgrades, and to avoid
    requiring a reboot when the kernel upgrades, you should keep this current with
    the latest daily Ubuntu AMI build. You will need to choose an AMI related to
    your chosen region. By default this is us-east-1
  - SSHKeyName: Select your SSH key from the dropdown. For your first trial run
    you will use this key to ssh in - but in the Next Steps section of this
    document we will move to individual accounts.
  - SSHLocation: The IP address range that you want SSH to be allow in from. This
    configures AWS security groups to limit incoming SSH access. Leave it as 0.0.0.0/0
    for a global allow for this test.
  - SSLCertificateIdArn: The Arn of the SSL key you generated earlier. You can
    find instructions for locating it above if needs be.
  - WebServerCountDesired / WebServerCountMax / WebServerCountMin: The number
    of entries in the WebServer auto scaling group. See http://docs.aws.amazon.com/AutoScaling/latest/DeveloperGuide/as-maintain-instance-levels.html
    for details. Leave this as 1/1/1 for this demo.
  - WebServerHealthCheckType: Set this to EC2 for now - we'll cover it
    in more detail below.
  - WebserverInstanceType: Which EC2 server to use for webservers. We've had
    success with the c4.xlarge instance type - but you can use any SSD-HVM
    supporting instance here if you want. Note that the smaller you go, the
    longer the systems take to start and build.
6. Click "Next", and "Next" again to go through the advanced options.
7. Acknowledge the warning about the creation of IAM resources. These Roles and
    permissions are detailed in the stack.json file, along with related URLs
    in the comment section if you are concerned.
8. Click "Create"
9. Wait for the stack to build. If everything is successful, it should
    change to state 'CREATE_COMPLETE'.

Once the stack is built, you should have one or more servers in the EC2 console.

Additionally, you should see Elastic Load Balancers and auto scaling groups in
the EC2 console.


## Create CodeDeploy Configs

Once the app has been built, create a CodeDeploy configuration.

Log into the AWS web console and make sure you are in the 'Ireland'
(us-east-1) region. Then:

1. Services -> CodeDeploy
2. Create a new CodeDeploy config ("Get Started Now")
3. Choose "Custom Deployment" and "Skip Walkthrough", since you don't want ot install the demo app
4. Set Application Name to "blog-yourname-production", "blog-yourname-preview",
    "blog-yourname-dev", or similar (note that this is case-sensitive, and the
    deployment script expects an exact match to be able to do the deploy.
5. Set the Deployment Group Name to "RailsAppServers" (again - case and naming is important so that the
    deploy scripts can locate this item correctly!)
6. In "Add Instances", change the Tag Type to "Auto Scaling Group", and add the
    correct WebServerGroup for the env you are working on (eg:
    staging-WebServerGroup-something-here)
7. Set the "Deployment Config" to be "CodeDeployDefault.OneAtATime" (the default)
8. Set the "Service Role ARN" to the CodeDeployTrustRole of your environment
    (eg: staging-CodeDeployTrustRole-something-here). Take care not to select the InstanceRole!
9. Click "Create Application"

## Deploy the App

Next, you'll need to upload the application itself. To do this, you need a
checked-out git copy of http://github.com/unboxed/aws-ansible-autoscaling-and-code-deploy-app

```shell
$ # These should be set from earlier:
$ export AWS_DEFAULT_REGION='us-east-1'
$ export AWS_ACCESS_KEY_ID='AKI...'
$ export AWS_SECRET_ACCESS_KEY='...'
$ # Upload the app itself - note that you must pass the parameter that matches
$ # the parameters you supplied to the S3 bucket stack.
$ # Note that we supply a branch or tag or SHA to the push:
$ scripts/push-release-to-s3.sh bucketprefix blog dev HEAD
Copying fact files to S3
...
SUCCESS
```

Have a look on the CodeDeploy UI - you should see CodeDeploy rolling out the
new version of the codebase to your servers.

## Testing the App

To test the app, you'll either need to edit your /etc/hosts file, or set up
a DNS record as per http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/using-domain-names-with-elb.html

To locate the DNS name of the Load Balancer, go to the AWS console, select EC2,
select Load Balancers, and select the Load balancer. You will see "DNS Name"
listed in the bottom section - something like 'blog-somethingoverhere-123123123.us-east-1.elb.amazonaws.com'

If you want to test from the command line, grab the hostname of the Elastic
Load Balancer, and run curl. You should see the rails app response. You'll need
to make sure that the hostname matches the value you set in the
```webserver_hostname``` parameter of ```facts/dev/app.fact```

```
curl -k -H 'Host: blog-dev.example.com' https://blog-dev-123123123.us-east-1.elb.amazonaws.com
```

# Next Steps

## Switch to ELB Healthchecks

When you first create the stack, we default to "EC2 healthchecks". If we
defaulted to ELB healthchecks up front, we'd be in a catch-22. The servers
would come up without a webserver app (since we only create the deployment in
the following step).

As the app isn't running, the ELB would consider the servers to be down. It
then classify them as unhealthy. The autoscaling group would delete them and
create a new server in it's place. Servers would be created and destroyed until
you went through the "Deploy the App" step above.

Once you've got a working app up and running - you should change the healthcheck
to be an ELB healthcheck rather than an EC2 healthcheck.

To do this, Log into the AWS UI, go to the CloudFormation Service, and update
the Cloudformation stack, re-upload the stack file, and select ELB instead of
EC2. You should check the 'Use existing value' value is ticked for all items
that had passwords or similar.

This way, if your webserver stops responding, ELB will replace it.

## Configure Individual Accounts

You probably want individual account access, rather than logging in as ubuntu.

Note that if all goes well with the server build, you will not be able to log
in with that key - you should instead use your own configured username and key
from the ```ansible/group_vars/all``` file

# Diagnosing Problems

If your server doesn't come up correctly, you can check the logs by sshing in
as ubuntu and viewing /var/log/cloud-init.log

If you are having problems with an existing system, you can view the system
logs via the AWS CloudWatch interface. Log in, click on the CloudWatch service,
select "Logs", select the type of log and then the instance ID.

# A Word of Warning

You should be very careful when updating the CloudFormation stack. You should
take care to test your changes on a matching stack before making any changes
to production.

Additionally - you should strongly avoid making changes to the AWS
infrastructure other than through CloudFormation uploads.

It's possible that Cloudformation could try and update your stack config,
run into problems, and then revert to the previous config.

If you've made manual changes to the stack, your rolled-back version of the
stack won't be as you expect. This could lead to a double failure in
Cloudformation: The stack can't roll forward - but it also can't roll back.

In that case, your only option is to delete the stack! Doing so will delete
your RDS database too - AND it's backups. So you must be *VERY* careful to only
do this if you've made manual Postgres DB Backups, or are comfortable restoring
your DB from a manual RDS backup!
