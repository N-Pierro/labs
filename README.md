# labs
devops labs

## Project outline 

1.
    k8s/
├── redis-deployment.yaml
├── redis-service.yaml
├── web-deployment.yaml
├── web-service.yaml
├── web-configmap.yaml




2. Create aws ECR (aws container repository)
    - `aws ecr create-repository --repository-name project-a/sample-repo`
    Output:

    ```sh
    {
              "repository": {
                  "registryId": "123456789012",
                  "repositoryName": "project-a/sample-repo",
                  "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/project-a/sample-repo"
              }
          }
     ```
**Enable ECR image scanning**

Turn on enhanceing scanning:

```sh
aws ecr put-registry-scanning-configuration \
    --scan-type ENHANCED
```
This gives:

    - CVE detection 
    - Base image vulnerabilities
    - OS + library findings

see: aws ecr help

3. Build and push image to ecr
    
    - Firts Authenticate docker to ECR. This uses temporary auth token

        ```sh
        aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.eu-central-1.amazonaws.com
        ```

    build and tag image with docker: 

    `docker tag -t image-name:latest`

    Run docker image ls to get image(s) present in your local system:

    `docker image ls` 

    output:

    docker image ls

REPOSITORY   |                                                     TAG            |                                                           IMAGE ID |        CREATED |         SIZE |

flask-web |                                                                 dev |                                                                      443663845f9f |  46 seconds ago |  414MB |


Tag image for ecr:
     
    `ddkr.ecr.eu-central-1.amazonaws.com/project-devops/flask-web:latest`

check image:

`docker image ls` 

Push image to ecr:

    `docker push account-it.dkr.ecr.eu-central-1.amazonaws.com/project-devops/flask-web:latest`


Update kubernetes deployment to use ecr image:

    `image: your-image:latest`

example:

    `image: 123456789012.dkr.ecr.eu-central-1.amazonaws.com/flask-web:latest`



## Deploy to AWS EKS

AWS eks is managed aws cluster. The cluster is made of 2 parts: 

1. Control Plane 
2. Node Groups 

### Create eks cluster (CLI)

`eksctl create cluster --name flask-cluster --region eu-central-1 --nodegroup-name general-ng-v1 --node-type t3.small --nodes 2 `

### Delete cluster aggressively

` eksctl delete cluster --name my-cluster`

What this command does:

- Creates VPC
- Create subnets 
- Creates EKS control plane
- Create EC2 worker nodes
- Attaches IAM roles
- Configures kubeconfig

### Mental representation 

<details>
<summary><strong>EKS Cluster Architecture</strong></summary>

- **Control Plane (AWS-managed)**
  - API Server
  - Scheduler
  - etcd
- **Node Groups (your compute)**
  - EC2 instances (worker nodes)
  - Pods run here

</details>


3 Types of node groups:

1. Managed Node Group 

AWS manages:
    - AMI updates
    - node draining
    - scaling integration

You manage:
    - instance type
    - min/max/desired size 

This creates a node group (cli)

`eksctl create nodegroup --cluster my-cluster --name general-ng --node-type t3.medium --nodes 2 `

2. Self-managed Node group

You manage:

    - AMIs
    - upgrades
    - lifecycle

3. Fargete (no nodes at all)

    - No EC2 instances
    - Pods run serverlessly
    - No node groups

    Used for:

        - low ops overhead
        - stateless workloads

Eks node group is backed by:
- an EC2 autoscaling group
- a Launch Template 

### Why node groups exist

Different workloads have different needs 

Example:

workload                |        Node group
-------------------------------------------------
web apps                |       general-ng

Batch jobs              |       spot-ng

Databases               |       memory-ng

GPU workloads           |       gpu-ng

each node group can have:

    - different instance types
    - taint & labels
    - pricing model 

### Kubernetes - node groups

node groups expose nodes to kubernetes as normal nodes

`kubectl get nodes`

output example:

`ip-10-0-1-23.ec2.internal`
`ip-10-0-2-45.ec2.internal`

so when you run the command 

`kubectl apply -f path/to/file`

scheduler decides:
    - which node
    - from which node group
    - based on:
        - resource
        - labels
        - taints

    If you happen to delete a node group:
        - All nodes in the node group are terminated
        - Pods are rescheduled elsewhere 
        - If no other node groups exist -> pods stay Pending

### Common kubernetes troubleshooting and Debugging commands

check node pod capacity

`kubectl describe node <node-name> | grep -i pods`

output:

`pods: 4`
`pods: 11`
`Non-terminated Pods:   (4 in total)` This is important, it tells how much node resources is used by cluster(system pods).

By default kubernetes aws eks cluster creates:
- CoreDNS
- Metric Server
- Kube-proxy
- Amazon-vpc CNI

Check for namespaces:

`kubectl get ns` or `kubectl get namespace` or `kubectl get ns -o wide`

Show clusters that exist in AWS, regardless of kubeconfig

`aws eks list-cluster --region eu-central-1`

Describe a specific cluster:

`aws eks describe-cluster --name flask-cluster --region eu-central-1`

Verify the cluster is reachable:

`kubectl cluster-info`



## Troubleshoot cloudformation stack failure, when creating aws eks cluster 

1. Identify the failing CloudFormation stack

run:

```sh
aws cloudformation list-stacks --stack-status-filter CREATE_FAILED ROLLBACK_COMPLETE ROLLBACK_FAILED --region eu-central-1
```

Look for the stack name like:

```sh
{
    "StackSummaries": [
        {
            "StackId": "arn:aws:cloudformation:eu-central-1:account-id:stack/eksctl-flask-cluster-nodegroup-general-ng-v1/1wefe810-bd12-10b0-b83c-3571ba65ad97",
            "StackName": "eksctl-flask-cluster-nodegroup-general-ng-v1",
            "TemplateDescription": "EKS Managed Nodes (SSH access: false) [created by eksctl]",
            "CreationTime": "2026-01-11T21:27:18.957000+00:00",
            "DeletionTime": "2026-01-11T22:00:54.925000+00:00",
            "StackStatus": "ROLLBACK_COMPLETE",
            "DriftInformation": {
                "StackDriftStatus": "NOT_CHECKED"
            }
        }
    ]
```

copy the StackName

2. Get the exact failure reason (import)

Replace <StackName>:

```sh 
aws cloudformation describe-stack-events --stack-name <StackName> --region eu-central-1 --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceType,ResourceStatusReason]' --output table
```
output:

 ```sh
  ManagedNodeGroup|  AWS::EKS::Nodegroup |  Resource handler returned message: "[Issue(Code=AsgInstanceLaunchFailures, Message=Could not launch On-Demand Instances. InvalidParameterCombination - The specified instance type is not eligible for Free Tier. For a list of Free Tier instance types, run 'describe-instance-types' with the filter 'free-tier-eligible=true'. Launching EC2 instance failed., ResourceIds=[eks-general-ng-v1-61wefe810-bd12-10b0-b83c-3571ba65ad97])] (Service: null, Status Code: 0, Request ID: null)" (RequestToken: 1wefe810-bd12-10b0-b83c-3571ba65ad97, HandlerErrorCode: GeneralServiceException)  
 ```
Reson for failure:

`InvalidParameterCombination - The specified instance type is not eligible for Free Tier`




## Helm 

helm install team1-dev ./web-stack \
  -f values-dev.yaml \
  -n team1-dev \
  --create-namespace


### Render file to check it matches the kubernetes manifest file:

`helm template team1-dev . -f values-dev.yaml`

Dry-run against kubernetes: 

```sh
helm template . -f values-dev.yaml | kubectl apply --dry-run=client -f -
```
output: 

```sh
configmap/release-name-web-config created (dry run)
service/release-name-redis created (dry run)
service/release-name created (dry run)
deployment.apps/release-name-redis created (dry run)
deployment.apps/release-name-web created (dry run)
```

re-run: 

`helm upgrade --install team1-dev . -f values-dev.yaml -n team1-dev`


List release across all namespaces:

`helm list -A`

## Debug

To test from locally (open 2 terminals):

1. on one run: `kubectl port-forward service/team1-dev 8000:8000 -n team1-dev`
2. on the other run: `curl http://localhost:8000/health`


<details>
<summary><strong> Phase 4 (CI/CD) end-t0-end automation </strong></summary>

- **CI/CD with Github Actions (ECR -> EKS -> Helm)**

    - Build the Flask image 
    - Pushes it to Amazon ECR
    - Deploy it to EKS using Helm
    - Requires no manual kubectl or helm command 

            git push
    
               ↓

           GitHub Actions
    
                ↓
    
            Docker build
    
                ↓
    
            Amazon ECR
    
                ↓
    
            Helm upgrade
    
                ↓
    
               EKS


</details>