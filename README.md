### Repository for road show commands/instructions

1.) Have a clean AWS account
2.) Use either Cloud9 or your local cli #ensure you have administrative privledges
3.) Run the pre-req.sh to install all of the needed dependencies

eks
- provision a cluster with eksctl and new config file format
- show the dashboard
- demonstrate HPA (ideally with SQS queue depth)
- demonstrate cluster autoscaling
- demonstrate ci/cd with codesuite

fargate
- provision a fargate cluster with ecs-cli
 
cloudmap
- register service with discivery
- demo http service discovery
- demo dns service discovery

demo appmesh
- color application
- deploy some services in EKS
- deploy at least one service in ECS

demo firecracker
- bare metal instance

#### clone the repository
```
https://github.com/omarlari/roadshow-demo.git
```
#### install tooling
```
chmod u+x ~/environment/roadshow-demo/pre-reqs.sh
```

```
~/environment/roadshow-demo/pre-reqs.sh
```

### eks demo

#### create basic cluster with eksctl

```
eksctl create cluster -f ~/environment/eks/cluster.yaml
```

#### issue basic commands

```
kubectl get nodes

kubectl get ns

kubectl get pods --all-namespaces
```

#### demonstrate basic service deployment

```
kubectl run hello-world --replicas=5 --labels="run=load-balancer-example" --image=nginx --port=80

kubectl get deployments hello-world

kubectl describe deployments hello-world

kubectl expose deployment hello-world --type=LoadBalancer --name=my-service

kubectl get services my-service -o wide
```

#### Helm install

```
cd ~/environment

curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

chmod +x get_helm.sh

./get_helm.sh

kubectl apply -f ~/environment/roadshow-demo/eks/rbac-helm.yaml

helm init --service-account tiller

helm update
```

#### HPA

```
helm install stable/metrics-server --name metrics-server --version 2.4.0 --namespace metrics

kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

Deploy a sample application:

```
kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
```

Create HPA resource:

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

kubectl get hpa
```

generate load:

```
kubectl run -i --tty load-generator --image=busybox /bin/sh
while true; do wget -q -O - http://php-apache; done

kubectl get hpa -w
```

clean up
remove the load generator deployment

#### Cluster autoscaler

add your specifc ASG to the ~/environment/roadshow-demo/cluster_autoscaler.yml file

```
kubectl apply -f ~/enable/roadshow-demo/cluster_autoscaler.yml

kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

#### Codesuite integration

### Fargate

#### create an ec2 keypair (needed for ec2 mode, appmesh only runs on ec2 mode)

```
aws ec2 create-key-pair --key-name roadshow-<region>
```

#### create task execution role

```
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json
```

#### attach the policy

```
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

#### configure ecscli

```
ecs-cli configure --cluster roadshow --region us-west-2 --default-launch-type EC2 --config-name roadshow

ecs-cli configure profile --access-key --secret-key --profile-name roadshow
```

#### bring up cluster

```
ecs-cli up --keypair ~/environment/roadshow-demo/roadshow-pdx --capability-iam --size 2 --instance-type t2.small --cluster-config roadshow
```

#### Deploy ALB Cloudformation

```
aws cloudformation create-stack --stack-name ecs-demo-alb --template-body file://~/environment/roadshow-demo/fargate/alb-external.yml --parameters ParameterKey=EcsVpc,ParameterValue=vpc-0397f4a6f73468c08 ParameterKey=Subnet1,ParameterValue=subnet-04f397d24c6751d11 ParameterKey=Subnet2,ParameterValue=subnet-059d75daadf909619 ParameterKey=EcsSecurityGroup,ParameterValue=sg-01670bfbd7046e454
```

#### Deploy sample app to ALB

```
export target_group_arn=$(aws cloudformation describe-stack-resources --stack-name ecs-demo-alb | jq -r '.[][] | select(.ResourceType=="AWS::ElasticLoadBalancingV2::TargetGroup").PhysicalResourceId')

ecs-cli compose --project-name ecsdemo-frontend service up \
    --create-log-groups \
    --target-group-arn $target_group_arn \
    --private-dns-namespace service \
    --enable-service-discovery \
    --container-name apache \
    --container-port 80 \
    --vpc $vpc \
    --launch-type FARGATE
```

#### register and scale the service

```
cd ~/environment/roadshow-demo/cloudmap/service_one

ecs-cli compose --project-name apache service up --private-dns-namespace roadshow --vpc vpc-01fe75ea2b315621e --enable-service-discovery --create-log-groups --launch-type FARGATE

ecs-cli ps

ecs-cli compose --project-name apache service scale 3

ecs-cli ps

aws servicediscovery discover-instances --namespace-name roadshow --service-name apache --region us-east-2
```

### Appmesh

Create the mesh:

```
aws appmesh create-mesh --mesh-name APP_MESH_DEMO
```

Virtual Nodes:
```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "serviceName": "colorgateway.default.svc.cluster.local"
            }
        },
        "backends": [
            "colorteller.default.svc.cluster.local"
        ]
    },
    "virtualNodeName": "colorgateway-vn"
}'
```

```

aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "serviceName": "colorteller.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-vn"
}'

```

```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "serviceName": "colorteller-black.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-black-vn"
}'
```

```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "serviceName": "colorteller-blue.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-blue-vn"
}'
```

```
aws appmesh create-virtual-node  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "listeners": [
            {
                "portMapping": {
                    "port": 9080,
                    "protocol": "http"
                }
            }
        ],
        "serviceDiscovery": {
            "dns": {
                "serviceName": "colorteller-red.default.svc.cluster.local"
            }
        }
    },
    "virtualNodeName": "colorteller-red-vn"
}'
```

Create the routers:
```
aws appmesh create-virtual-router  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "serviceNames": [
            "colorgateway.default.svc.cluster.local"
        ]
    },
    "virtualRouterName": "colorgateway-vr"
}'
```

```
aws appmesh create-virtual-router  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "spec": {
        "serviceNames": [
            "colorteller.default.svc.cluster.local"
        ]
    },
    "virtualRouterName": "colorteller-vr"
}'
```

Create the routes:
```
aws appmesh create-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "routeName": "colorgateway-route",
    "spec": {
        "httpRoute": {
            "action": {
                "weightedTargets": [
                    {
                        "virtualNode": "colorgateway-vn",
                        "weight": 100
                    }
                ]
            },
            "match": {
                "prefix": "/"
            }
        }
    },
    "virtualRouterName": "colorgateway-vr"
}'
```

```
aws appmesh create-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
   "routeName": "colorteller-route",
   "spec": {
       "httpRoute": {
           "action": {
               "weightedTargets": [
                   {
                       "virtualNode": "colorteller-vn",
                       "weight": 1
                   }
               ]
           },
           "match": {
               "prefix": "/"
           }
       }
   },
   "virtualRouterName": "colorteller-vr"
}'
```

deploy the app:

```
kubectl apply -f https://raw.githubusercontent.com/geremyCohen/colorapp/master/colorapp.yaml
```

deploy the curler

```
kubectl run -it curler --image=tutum/curl /bin/bash
```

run fetch app:
```
kubectl run -it curler --image=tutum/curl /bin/bash
```

run command:
```
while [ 1 ]; do  curl -s --connect-timeout 2 http://colorgateway.default.svc.cluster.local:9080/color;echo;sleep 1; done
```

Modify route:

```
aws appmesh update-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "routeName": "colorteller-route",
    "spec": {
        "httpRoute": {
            "action": {
                "weightedTargets": [
                    {
                            "virtualNode": "colorteller-blue-vn",
                            "weight": 3
                        },
                        {
                            "virtualNode": "colorteller-red-vn",
                            "weight": 3
                        },
                        {
                            "virtualNode": "colorteller-black-vn",
                            "weight": 3
                        }
                ]
            },
            "match": {
                "prefix": "/"
            }
        }
    },
    "virtualRouterName": "colorteller-vr"
}'
```

Modify route again for 50/50:

```
aws appmesh update-route  --mesh-name APP_MESH_DEMO --cli-input-json '{
    "routeName": "colorteller-route",
    "spec": {
        "httpRoute": {
            "action": {
                "weightedTargets": [
                        {
                            "virtualNode": "colorteller-red-vn",
                            "weight": 5
                        },
                        {
                            "virtualNode": "colorteller-black-vn",
                            "weight": 5
                        }
                ]
            },
            "match": {
                "prefix": "/"
            }
        }
    },
    "virtualRouterName": "colorteller-vr"
}'
```

### Firecracker

aws ec2 run-instances --image-id ami- --count 1 --instance-type i3.metal --key-name roadshow-pdx








