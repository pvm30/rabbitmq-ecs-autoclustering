Playing with Arnaud. I had to add the proxies to the Docker file
-------------------------------------------------------------------------------------

ENV http_proxy http://10.108.61.254:8080
ENV https_proxy http://10.108.61.254:8080

and to my sessions

set HTTP_PROXY=http://10.108.61.254:8080
set HTTPS_PROXY=http://10.108.61.254:8080

docker build -t arnaud/rabbitmq-asg-autocluster .

aws ecr get-login --no-include-email
docker login...
docker tag arnaud/rabbitmq-asg-autocluster:latest 563290296137.dkr.ecr.eu-west-1.amazonaws.com/pvm14/rabbitmq-asg-autocluster:latest
docker push 563290296137.dkr.ecr.eu-west-1.amazonaws.com/pvm14/rabbitmq-asg-autocluster:latest

I can run it locally like this

docker run -d --hostname localhost-rabbit-arnaud --name arnaud -p 15672:15672 -e AWS_ASG_AUTOCLUSTER=false arnaud/rabbitmq-asg-autocluster

I created manually a cluster from my aws console named  'rabbitmq-cluster-created-from-console' with two instances

I created my policy  arn:aws:iam::563290296137:policy/Rabbitmq-ECS-Auto-Clustering and I attached it to 'ecsInstanceRole'

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingInstances",
                "ec2:DescribeInstances",
                "autoscaling:DescribeAutoScalingGroups"
            ],
            "Resource": "*"
        }
    ]
}

and it seems that to run locally run the container on cluster mode

docker run --hostname localhost-rabbit-arnaud --name arnaud -p 15672:15672 -e AWS_ASG_AUTOCLUSTER=true -e RABBITMQ_ERLANG_COOKIE=ALWEDHDBZTQYWTJGTXWV arnaud/rabbitmq-asg-autocluster

I created the task definition rabbitmq-ecs-autoclustering-task-definition and I tried to run it but it stopped

aws ecs run-task --task-definition rabbitmq-ecs-autoclustering-task-definition --cluster rabbitmq-cluster-created-from-console --count 1

The only fucking way to find out what happened was issuing this command:


C:\PVM14\WIPO\TASKS-JIRAS\HCS-367 Achieving HA for RabbitMQ in Amazon EC2\rabbitmq-ecs-autoclustering\docker-image (master)
λ aws ecs describe-tasks --cluster rabbitmq-cluster-created-from-console --task 0ea372c9-7da3-442a-88a5-a540f8c0e9ac
{
    "failures": [],
    "tasks": [
        {
            "taskArn": "arn:aws:ecs:eu-west-1:563290296137:task/0ea372c9-7da3-442a-88a5-a540f8c0e9ac",
            "group": "family:rabbitmq-ecs-autoclustering-task-definition",
            "attachments": [],
            "executionStoppedAt": 1520250699.0,
            "overrides": {
                "containerOverrides": [
                    {
                        "name": "rabbit"
                    }
                ]
            },
            "launchType": "EC2",
            "lastStatus": "STOPPED",
            "containerInstanceArn": "arn:aws:ecs:eu-west-1:563290296137:container-instance/78b94f65-1cb2-48f1-afe4-a43d2ecb1957",
            "createdAt": 1520250694.385,
            "stoppingAt": 1520250699.061,
            "version": 2,
            "clusterArn": "arn:aws:ecs:eu-west-1:563290296137:cluster/rabbitmq-cluster-created-from-console",
            "memory": "512",
            "desiredStatus": "STOPPED",
            "stoppedReason": "Essential container in task exited",
            "taskDefinitionArn": "arn:aws:ecs:eu-west-1:563290296137:task-definition/rabbitmq-ecs-autoclustering-task-definition:2",
            "cpu": "512",
            "containers": [
                {
                    "containerArn": "arn:aws:ecs:eu-west-1:563290296137:container/006e6c2a-4be7-4876-b1e7-692d29fa3bf7",
                    "taskArn": "arn:aws:ecs:eu-west-1:563290296137:task/0ea372c9-7da3-442a-88a5-a540f8c0e9ac",
                    "name": "rabbit",
                    "lastStatus": "STOPPED",
                    "reason": "CannotStartContainerError: API error (500): failed to initialize logging driver: ResourceNotFoundException: The specified log group does not exist.\n\tstatus code: 400, request id: 904a550f-206b-11e8-8e20-1363f96a7838\n",
                    "networkInterfaces": []
                }
            ],
            "stoppedAt": 1520250699.061
        }
    ]
}

I had to create a new image from an instance in Amazon without having those disgusting proxy problems but for some reason the cluster is still not being formed. Executing this command

root@ip-10-2-1-83:/# python /opt/rabbitmq_asg_autocluster.py

botocore.exceptions.ClientError: An error occurred (AccessDenied) when calling the DescribeAutoScalingInstances operation: User: arn:aws:sts::563290296137:assumed-role/ecsTaskE
xecutionRole/91768923-b871-4e21-b80f-0b1a18bc7a26 is not authorized to perform: autoscaling:DescribeAutoScalingInstances

So I'm going to provide ecsTaskExecutionRole with the additional authorizations it seems to need

The rest of the problems I had to solve were related to:

 1) connectivity between machines (ports 4369, 25672 must be mutually visible)
 2) docker image not being properly generated because of the proxies (finally I resorted to build the image in an EC2 instance)
 3) the python script using by default a different regions instead of  the one I'm using 'eu-west-1'
 