ecs-formation
==========

[![Circle CI](https://circleci.com/gh/openfresh/ecs-formation.svg?style=shield&circle-token=baf60b45ce2de8c5d11b3e6d77a3a23ebf2d5991)](https://circleci.com/gh/openfresh/ecs-formation)
[![Language](http://img.shields.io/badge/language-go-brightgreen.svg?style=flat)](https://golang.org/)
[![issues](https://img.shields.io/github/issues/openfresh/ecs-formation.svg?style=flat)](https://github.com/openfresh/ecs-formation/issues?state=open)
[![License: MIT](http://img.shields.io/badge/license-MIT-orange.svg)](LICENSE)

ecs-formation is a tool for defining several Docker continers and clusters on [Amazon EC2 Container Service(ECS)](https://aws.amazon.com/ecs/).

# Features

* Define services on ECS cluster, and Task Definitions.
* Supports YAML definition like docker-compose. Be able to run ecs-formation if copy docker-compose.yml(formerly fig.yml).
* Manage ECS Services and Task Definitions by AWS API.
* Support ELB(Classic Load Balancer) and ALB(Application Load Balancer)
* [Service Auto Scaling](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)
* [Task Placement Policy](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement.html)

# Usage

### Setup

#### Installation

ecs-formation is written by Go. Please run `go get`.

```bash
$ go get github.com/openfresh/ecs-formation
```

#### Define environment variables

ecs-formation use [cobra](https://github.com/spf13/cobra). It supports yaml configuration.

Prepare `~/.ecs-formation.yaml` for project configuration.

```Ruby
project_dir: ~/your_project_dir/
aws_region: us-east-1
```

ecs-formation requires environment variables to run, as follows.

* AWS_ACCESS_KEY: AWS access key
* AWS_SECRET_ACCESS_KEY: AWS secret access key
* AWS_REGION: Target AWS region name

#### Make working directory

Make working directory for ecs-formation. This working directory should be managed by Git.

```bash
$ mkdir -p path-to-path/test-ecs-formation
$ mkdir -p path-to-path/test-ecs-formation/task
$ mkdir -p path-to-path/test-ecs-formation/service
$ mkdir -p path-to-path/test-ecs-formation/bluegreen
```

### Manage Task Definition and Services

#### Make ECS Cluster

You need to create ECS cluster in advance. And also, ECS instance must be join in ECS cluster.

#### Define Task Definitions

Make Task Definitions file in task directory. This file name is used as ECS Task Definition name.

```bash
(path-to-path/test-ecs-formation/task) $ vim test-definition.yml
nginx:
  image: nginx:latest
  ports:
    - 80:80
  environment:
    PARAM1: value1
    PARAM2: value2
  links:
    - api
  memory_reservation: 256
  cpu_units: 512
  essential: true

api:
  image: your_namespace/your-api:latest
  ports:
    - 8080:8080
  memory: 1024
  cpu_units: 1024
  essential: true
  links:
    - redis

redis:
  image: redis:latest
  ports:
    - 6379:6379
  memory: 512
  cpu_units: 512
  essential: true
```

#### Define Services on Cluster

Make Service Definition file in cluster directory. This file name must be equal ECS cluster name.

For example, if target cluster name is `test-cluster`, you need to make `test-cluster.yml`.

```bash
(path-to-path/test-ecs-formation/service) $ vim test-cluster.yml
test-service:
  task_definition: test-definition
  desired_count: 1
  role: your-ecs-elb-role
  load_balancers:
    -
      name: test-elb
      container_name: nginx
      container_port: 80
  autoscaling:
    target:
      min_capacity: 0
      max_capacity: 1
      role: arn:aws:iam::your_account_id:role/ecsAutoscaleRole

```

In case of ALB, you should specify `target_group_arn`.
```bash
(path-to-path/test-ecs-formation/service) $ vim test-cluster.yml
test-service:
  task_definition: test-definition
  desired_count: 1
  role: your-ecs-elb-role
  load_balancers:
    -
      target_group_arn: test-target-group-arn
      container_name: nginx
      container_port: 80
```



#### Keep desired_count at updating service

If you modify value of `desired_count` by AWS Management Console or aws-cli, you'll fear override value of `desired_count` by ecs-formation. This value should be flexibly changed in the operation.

If `keep_desired_count` is `true`, keep current `desired_count` at updating service.

```bash
(path-to-path/test-ecs-formation/service) $ vim test-cluster.yml
test-service:
  task_definition: test-definition
  desired_count: 1
  keep_desired_count: true
```

#### Task Placement Policy

Supports `placement_constraints` and `placement_strategy`. If you update these options, it's necessary to rebuild service once.    

```bash
(path-to-path/test-ecs-formation/service) $ vim test-cluster.yml
test-service:   
  task_definition: test-definition
  desired_count: 1
  keep_desired_count: true
  placement_constraints:
    - type: memberOf
      expression: "attribute:ecs.instance-type =~ t2.*"
  placement_strategy:
    - type: spread
      field: "attribute:ecs.availability-zone"
```


#### Manage Task Definitions

Show update plan.

```bash
(path-to-path/test-ecs-formation $ ecs-formation task plan --all
(path-to-path/test-ecs-formation $ ecs-formation task plan -t test_definition
```

Apply all definition.

```bash
(path-to-path/test-ecs-formation $ ecs-formation task apply --all
(path-to-path/test-ecs-formation $ ecs-formation task apply -t test_definition
```

#### Manage Services on Cluster

Show update plan. Required cluster.

```bash
(path-to-path/test-ecs-formation $ ecs-formation service plan -c test-cluster --all
```

Apply all services.

```bash
(path-to-path/test-ecs-formation $ ecs-formation service apply -c test-cluster --all
```

Specify cluster and service.

```bash
(path-to-path/test-ecs-formation $ ecs-formation service apply -c test-cluster -s test-service
```

### Blue Green Deployment

ecs-formation supports blue-green deployment.

#### Requirements on ecs-formation

* Requires two ECS cluster. Blue and Green.
* Requires two ELB. Primary ELB and Standby ELB.
* ECS cluster should be built by EC2 Autoscaling group.

#### Define Blue Green Deployment

Make management file of Blue Green Deployment file in bluegreen directory.

```bash
(path-to-path/test-ecs-formation/bluegreen) $ vim test-bluegreen.yml
blue:
  cluster: test-blue
  service: test-service
  autoscaling_group: test-blue-asg
green:
  cluster: test-green
  service: test-service
  autoscaling_group: test-green-asg
primary_elb: test-elb-primary
standby_elb: test-elb-standby
```

Show blue green deployment plan.

```bash
(path-to-path/test-ecs-formation $ ecs-formation bluegreen plan
```

Apply blue green deployment.

```bash
(path-to-path/test-ecs-formation $ ecs-formation bluegreen apply -g test-bluegreen
```

if with `--nodeploy` option, not update services. Only swap ELB on blue and green groups.

```bash
(path-to-path/test-ecs-formation $ ecs-formation bluegreen apply --nodeploy -g test-bluegreen
```

If autoscaling group have several different classic ELB, you should specify array property of `chain_elb`. ecs-formation can swap `chain_elb` ELB group with main ELB group at the same time.

```Ruby
(path-to-path/test-ecs-formation/bluegreen) $ vim test-bluegreen.yml
blue:
  cluster: test-blue
  service: test-service
  autoscaling_group: test-blue-asg
green:
  cluster: test-green
  service: test-service
  autoscaling_group: test-green-asg
primary_elb: test-elb-primary
standby_elb: test-elb-standby
chain_elb:
  - primary_elb: test-internal-elb-primary
    standby_elb: test-internal-elb-standby
```

In case of ALB(Application Load Balancer), as follows.

```Ruby
(path-to-path/test-ecs-formation/bluegreen) $ vim test-bluegreen.yml
blue:
  cluster: test-blue
  service: test-service
  autoscaling_group: test-blue-asg
green:
  cluster: test-green
  service: test-service
  autoscaling_group: test-green-asg
elbv2:
  target_groups:
    - primary_group: test-internal-primary
      standby_group: test-internal-default
```

### Others
#### Passing custom parameters

You can use custom parameters. Define parameters in yaml file(task, service, bluegreen) as follows.

```Ruby
nginx:
    image: openfresh/nginx:${NGINX_VERSION}
    ports:
        - 80:${NGINX_PORT}
```

You can set value for these parameters by using `-p` option.

```bash
ecs-formation task -p NGINX_VERSION=1.0 -p NGINX_PORT=80 plan -t your-web-task
```

Also, support default parameter value.

```Ruby
nginx:
    image: openfresh/nginx:${NGINX_VERSION|latest}
    ports:
        - 80:${NGINX_PORT|80}
```

#### env_file

You can use `env_file` like docker-compose. https://docs.docker.com/compose/compose-file/#env-file

```Ruby
nginx:
    image: openfresh/nginx:${NGINX_VERSION}
    ports:
        - 80:${NGINX_PORT}
    env_file:
        - ./test1.env
        - ../test2.env
```

License
===
See [LICENSE](LICENSE).

Copyright © FRESH!. All Rights Reserved.
