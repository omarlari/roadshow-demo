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
- sample app 

demo firecracker
- bare metal instance

#### clone the repository
https://github.com/omarlari/roadshow-demo.git

#### install tooling
chmod u+x ~/environment/roadshow-demo/pre-reqs.sh
~/environment/roadshow-demo/pre-reqs.sh

### eks demo

#### create basic cluster with eksctl

eksctl create cluster -f ~/environment/eks/cluster.yaml

#### issue basic commands

kubectl get nodes

kubectl get ns

kubectl get pods --all-namespaces

#### demonstrate basic service deployment



#### Helm install

cd ~/environment

curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh

chmod +x get_helm.sh

./get_helm.sh

kubectl apply -f ~/environment/roadshow-demo/eks/rbac-helm.yaml

helm init --service-account tiller

helm update

#### HPA

helm install stable/metrics-server --name metrics-server --version 2.4.0 --namespace metrics

kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

Deploy a sample application:
kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80

Create HPA resource:
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

kubectl get hpa

generate load:
kubectl run -i --tty load-generator --image=busybox /bin/sh
while true; do wget -q -O - http://php-apache; done

kubectl get hpa -w

clean up
remove the load generator deployment

#### HPA with SQS


#### Cluster autoscaler

add your specifc ASG to the ~/environment/roadshow-demo/cluster_autoscaler.yml file

kubectl apply -f ~/enable/roadshow-demo/cluster_autoscaler.yml

kubectl logs -f deployment/cluster-autoscaler -n kube-system

#### Codesuite integration




### Fargate

#### create an ec2 keypair (needed for ec2 mode, appmesh only runs on ec2 mode)

aws ec2 create-key-pair --key-name roadshow

#### create task execution role

aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json

#### attach the policy

aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

#### configure ecscli

ecs-cli configure --cluster roadshow --region us-east-2 --default-launch-type EC2 --config-name roadshow

ecs-cli configure profile --access-key --secret-key --profile-name roadshow

#### bring up cluster

ecs-cli up --keypair roadshow --capability-iam --size 2 --instance-type t2.small --cluster-config roadshow

#### register and scale the service

cd ~/environment/service_one

ecs-cli compose --project-name apache service up --private-dns-namespace roadshow --vpc vpc-01fe75ea2b315621e --enable-service-discovery --create-log-groups --launch-type FARGATE

ecs-cli ps

ecs-cli compose --project-name apache service scale 3

ecs-cli ps

aws servicediscovery discover-instances --namespace-name roadshow --service-name apache --region us-east-2









