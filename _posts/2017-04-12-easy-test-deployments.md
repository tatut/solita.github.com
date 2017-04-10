---
layout: post
title: Easily deploying test environments from CI builds with Travis, AWS and Slack
author: tatut
excerpt: Easy setup of test environments is important for feature branch based development. This post shows how to automatically deploy a new test environment with a simple chat interface.
tags:
- CI
- AWS
- cloud
- slack
---

Our project uses a git flow process: a new branch is created for each new feature. The feature is
developed in its entirety in the branch. This includes tests, database migrations and the code
itself. When the feature is ready, a GitHub pull request is made and another developer tests and
reviews the code.

We wanted to make testing another branch easy for developers. Developers will most likely have
their own branch in progress on their local machine and we wanted a system where developers
don't need to hop between their own branch and another one. Initially we made six development
test machines where the initial developer would deploy a new version for testing but this was
cumbersome as it needed manual deployment work and a reservation system to reserve a test
machine for your branch. Also the machines were idling most of the time.

## Along comes AWS

We are using the excellent [Travis CI](https://travis-ci.org/) for building our project and
running the automated test suites. We had the idea that we could automatically push the build
artifacts to S3 so that the latest build of a branch (along with a database dump) is always
available for deployment. Luckily Travis has direct support for S3 so the configuration is
a simple snippet in `.travis.yml` file:

```yml
deploy:
  provider: s3
  region: eu-central-1
  skip_cleanup: true
  bucket: "harjatravis"
  acl: public_read
  local_dir: s3-deploy
  on:
    all_branches: true
```

This instructs Travis to push all branches to our bucket. We set the bucket configuration
to automatically expire the artifacts after 2 weeks so that we don't accumulate old builds
for too long. Our build yields two artifacts: the production build .jar file and a PostgreSQL
dump of the test data that has been migrated to the latest version. Both are uploaded with
the branch name as part of the file name. Our branch names are usually the same as the JIRA
issue number so they are predictable.

## Setting up the build environment

Next we had to set up the actual machine for running the builds. We started with a
[CentOS](https://www.centos.org/) Linux image and started a new EC2 instance from it.
We then provisioned it with all the pieces needed to run the builds with
[Ansible](https://www.ansible.com). This includes installing NGINX, Java, PostgreSQL and
some shell scripts to start our service. Normally a production environment wouldn't have
the front-end proxy, the application service and the database on the same machine but for
testing purposes it will suffice.

After the machine had everything we need, we stopped it and created a new AMI from it to
use as a template for a deployed build.

## A serverless solution for starting up servers

Next we needed some way to easily start new EC2 machines for builds. AWS Lambda provides
a nice way to run predefined code and hook it up to services.

Lambda is described as a "serverless" solution for building applications and has a pay-per-execution
billing model. In our case, we are using Lambda to start new servers, so I guess the serverless
term doesn't really apply here.

We used Python as it can be easily edited right from the AWS Lambda console and has a good library
for using AWS services programmatically available. The library is called
[boto3](https://github.com/boto/boto3) and it can be used with zero configuration
from Lambda Python code.

```python
# coding=utf-8
import os
import boto3
import urllib2
import time
import ssl

initscript = '''...cloud config script here...'''

def check_branch(branch):
    req = urllib2.Request('...our deployment s3 bucket public url.../harja-travis-' + branch + '.jar')
    req.get_method = lambda: 'HEAD'
    try:
        res = urllib2.urlopen(req)
        if res.getcode() == 200:
            return True
    except:
        pass
    return False

def deploy(branch):
    script = initscript.replace('$BRANCH',branch)
    ec2 = boto3.client('ec2',region_name='eu-central-1')
    res = ec2.run_instances(ImageId='...ami-id...',
                            InstanceType='t2.medium',
                            UserData=script,
                            KeyName='...keypair...',
                            MinCount=1,
                            MaxCount=1)
    id = res['Instances'][0]['InstanceId']
    host = None
    while host is None:
        time.sleep(2)
        instances = ec2.describe_instances(InstanceIds=[id])
        for r in instances['Reservations']:
            for i in r['Instances']:
                dns = i['PublicDnsName']
                if dns != '':
                    host = dns
    return host

def wait_for_url(url):
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    status = None
    while status != 200:
        time.sleep(2)
        try:
            status = urllib2.urlopen(url, context=ctx).getcode()
        except:
            pass

def lambda_handler(event, context):
    branch = event['branch']
    txt = None
    if check_branch(branch):
        host = deploy(branch)
        url = 'https://' + host + '/'
        wait_for_url(url)
        txt = 'Started ' + branch + ': ' + url;
    else:
        txt = 'No Travis build for branch: ' + branch + '. Check build.'
    try:
        payload = '{"text": "'+txt+'"}'
        urllib2.urlopen(event['response_url'], payload)
    except Exception, e:
        print 'error sending Slack response: ' + str(e)
```

The above script does quite a lot. It takes an event that has two fields: branch and
response_url. The branch is the name of the git branch to deploy and response_url is
the Slack webhook URL to send a message to when the build is ready.

First the branch is checked by doing a HEAD request for the build artifact. If it doesn't exist,
bail out and notify the user that the build cannot be found.

If the build is ok, start a new EC2 instance for it from our previously created AMI and pass it a
user data cloud init script. More on cloud init later.
After the machine has been started we repeatedly call describe until the machine has a public
network name and return it.

Finally, we wait for the machine to be actually up and running the software before
notifying the user with a URL to the new machine. Running status is determined simply
by doing an HTTP GET request to it (with fake SSL, because we don't use real certificates
for the ephemeral test machines).

## Cloud init

[Cloud init](https://cloud-init.io/) is a simple way to customize servers that have been cloned from
a template. With EC2 Centos images, we can provide a cloud init script with the instance user data.

Cloud init is a YAML configuration file which can do lots of different customization tasks on the
starting instance. In our case, we only need to run some shell commands:

```yaml
#cloud-config
runcmd:
  - echo "Start PostgreSQL 9.5";
  - sudo service postgresql-9.5 start;
  - sudo -u postgres psql -c 'create database harja;';
  - sudo wget https://...s3 bucket url.../harja-travis-$BRANCH.pgdump.gz;
  - sudo wget https://...s3 bucket url.../harja-travis-$BRANCH.jar;
  - sudo zcat harja-travis-$BRANCH.pgdump.gz | sudo -u postgres psql harja;
  - sudo service nginx start
  - /home/centos/harja.sh $BRANCH
```

We start up PostgreSQL, create the database and restore it from the downloaded dump
and start nginx and our application service from a downloaded .jar file.

The above script is included in the Python lambda script and the `$BRANCH` is replaced with the
actual branch name parameter.