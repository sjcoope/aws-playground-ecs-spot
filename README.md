# ECS Web App - Instance Provision Learning
This is a very very basic sample project to help me understand how ECS Capacity Providers work and how they use Auto Scaling Groups to ensure sufficient instance capacity for tasks being run in an ECS service.

An overview of the architecture can be seen in the following diagram:

## Steps required to perform the test
The following commands are provided for testing purposes to create the AWS resources required for this test. **Beware** This can be an expensive test if the resources are not cleaned up after the test.

### Step 1: Create launch template with template file
---
```
aws ec2 create-launch-template \
    --cli-input-json file://lt-webapp.json
```

### Step 2: Create ASG and ARN Reference for Spot
---
```
aws autoscaling create-auto-scaling-group --cli-input-json file://asg-spot.json \
    --auto-scaling-group-name asg-spot --min-size 0 --max-size 10 \
    --vpc-zone-identifier "subnet-03c57bfd6a86fbe99,subnet-014a8649ac61512b2,subnet-08706aa9e697277db"

arn_spot=$(aws autoscaling  describe-auto-scaling-groups \
    --auto-scaling-group-name asg-spot --output json \
    --query 'AutoScalingGroups[0].AutoScalingGroupARN')
```

### Step 3: Create the Capacity Provider for Spot
---
```
aws ecs create-capacity-provider \
    --name "cp-spot" \
    --auto-scaling-group-provider "autoScalingGroupArn=$arn_spot,\
    managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,\
    maximumScalingStepSize=100}"
```

### Step 4: Create the ASG and ARN Reference for On-Demand
---
```
aws autoscaling create-auto-scaling-group --cli-input-json file://asg-od.json \
    --auto-scaling-group-name asg-od --min-size 0 --max-size 5 \
    --vpc-zone-identifier "subnet-03c57bfd6a86fbe99,subnet-014a8649ac61512b2,subnet-08706aa9e697277db"

arn_od=$(aws autoscaling  describe-auto-scaling-groups \
    --auto-scaling-group-name asg-od --output json \
    --query 'AutoScalingGroups[0].AutoScalingGroupARN')
```

### Step 5: Create the Capacity Provider for On-Demand
---
```
aws ecs create-capacity-provider \
    --name "cp-od" \
    --auto-scaling-group-provider "autoScalingGroupArn=$arn_od,\
    managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,\
    maximumScalingStepSize=100}"
```

### Step 6: Create ECS Cluster
---
```
aws ecs create-cluster \
    --cluster-name ecs-webapp \
    --capacity-providers cp-spot cp-od \
    --default-capacity-provider-strategy capacityProvider=cp-od,weight=1
```

### Step 7: Create ECS Service
---
```
aws ecs create-service \
    --capacity-provider-strategy capacityProvider=cp-od,base=2,weight=1 \
    capacityProvider=cp-spot,weight=1 \
    --cluster ecs-webapp \
    --service-name srv-web-app \
    --task-definition nginx \
    --desired-count 2 \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-03c57bfd6a86fbe99, subnet-014a8649ac61512b2, subnet-08706aa9e697277db]}"
```

## Clean Up
---

### Step 8: Remove ECS Service, ASG and Launch Template
---
```
# Delete ECS Service
aws ecs delete-service \
    --cluster ecs-webapp \
    --service srv-web-app --force

# Delete Auto Scaling Groups
aws autoscaling delete-auto-scaling-group \
    --auto-scaling-group-name asg-spot \
    --force-delete

aws autoscaling delete-auto-scaling-group \
    --auto-scaling-group-name asg-od \
    --force-delete

# Delete Launch Template
aws ec2 delete-launch-template --launch-template-name lt-webapp
```

### Step 9: Delete ECS Cluster
---
```
# Delete ECS Cluster
aws ecs delete-cluster --cluster ecs-webapp
```

### Step 10: Delete Capacity Providers
---
Has to be done after deleting the ECS cluster

```
# Delete Capacity Providers (has to be done after deleting the cluster)
aws ecs delete-capacity-provider --capacity-provider cp-spot

aws ecs delete-capacity-provider --capacity-provider cp-od
```