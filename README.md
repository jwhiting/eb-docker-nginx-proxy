# Elasic Beantalk Multi-container Docker Log Rotation Problem Example App

An example multi-container Docker Elastic Beanstalk application, based on the
official AWS example multi-container Docker app
[eb-docker-nginx-proxy](https://github.com/aws-samples/eb-docker-nginx-proxy),
which attempts to solve the problem of having all logs on the host rotated
regularly to avoid disk space exhaustion. 

## Goals:

1. Run a simple multi-container Docker application with as minimal changes to
   default Elastic Beanstalk configuration as is required.
2. Stream each container's stderr/stdout to CloudWatch log groups
2. Ensure that all logs on the host are rotated to prevent eventually filling
   the disk. Including in particular:
   1. `/var/lib/docker/containers/*/*-json.log` (the json log produced by the docker container)
   2. `/var/log/containers/*-stdouterr.log` (the non-json named container stdout/err log)

__Key problem:__ when a `/var/lib/docker/containers/*/*-json.log` file is rotated,
the corresponding `/var/log/containers/*-stdouterr.log` file stops being
appended with new log data. Since the latter is what is streamed to
CloudWatch, CloudWatch in turn stops receiving new log events.

## Requirements

The EC2 role in use by the Elastic Beanstalk instances must have permissions to
stream to CloudWatch. Elastic Beanstalk generates a role for you when you
create the environment, or, you can assign your own (the role is defined under
the "Security" card in the environment configuration UI). Either way, add the
following policy to the role in use via the IAM management console:

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:GetLogEvents",
                    "logs:PutLogEvents",
                    "logs:DescribeLogGroups",
                    "logs:DescribeLogStreams",
                    "logs:PutRetentionPolicy"
                ],
                "Resource": [
                    "*"
                ]
            }
        ]
    }

Also, the environment must be configured to stream logs to CloudWatch, which is
a checkbox under the "Software" card in the EB environment's configuration UI.

## Reproducing the problem

1. Launch an EB environment using a zip file of the contents of this repo, with
   the appropriate EC2 role permissions, and the CloutWatch streaming setting,
   per above.
2. Visit the endpoint URL homepage and static page once or twice and ensure you see the
   requests in CloudWatch in both the nginx and php container logs.
3. Wait an hour for the log rotation to take place. Or, instead of waiting an
   hour, trigger log rotation manually from a login shell on the EC2 host:

       sudo /usr/sbin/logrotate /etc/logrotate.elasticbeanstalk.hourly/logrotate.elasticbeanstalk.containers.conf
    
4. Repeat step 2 and see the that new log data no longer reaches CloudWatch
   (and, on the host itself, is no longer appended to the
   `/var/log/container/*-stdouterr.log` files)

## Modifications from the forked source project

* Added an `.ebextensions/log-rotation.conf` to define log rotation behaviors.
* Removed the nginx log volume (modified `dockerrun.aws.conf`) because its
  stdout/err is sent to CloudWatch instead.

