Create Kubernetes Cluster in AWS using kops
==============================================

What software I need to install, to create Kubernetes Cluster on AWS
	(1) kops
	(2) kubectl

-- Assumption
	(1) You are using mac or linux systems
	(2) You have an account with aws. If not, it is easy to get one. You will be charged only for the usage. When you are not working on the cluster, go ahead and delete the cluster

	
—install kops on mac
brew install kops

— install kubectl on mac
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl

You are ready to go....

— export few variables specific to your amazon account
export AWS_PROFILE=rdakshin
export AWS_ACCESS_KEY_ID=ProvideYourKeyIDValue
export AWS_SECRET_ACCESS_KEY=ProvideYourAccessKey
export AWS_DEFAULT_REGION=us-west-2 or any region of your choice


— create bucket the following buckets in aws
aws --profile rdakshin s3api create-bucket --bucket kubernetes-cluster-demo
— for data
aws --profile rdakshin s3api create-bucket --bucket kubernetes-aws-test-data
— for logs
aws s3api create-bucket --bucket kubernetes-aws-test-logs --region us-east-1

-- Create the cluster in gossip mode. This is to avoid having domain and sub-domains. The cluster name has to end as k8s.local

export KOPS_STATE_STORE=s3://kubernetes-cluster-demo
export NAME=myfirstcluster.k8s.local

— create cluster
kops create cluster     --zones us-west-2a     --name myfirstcluster.k8s.local  --state s3://kubernetes-cluster-demo --yes

— validate cluster
kops validate cluster --name myfirstcluster.k8s.local
— list nodes
kubectl get nodes --show-labels
kubectl get pods --all-namespaces --show-labels
— ssh to the master
ssh -i ~/.ssh/id_rsa admin@masterspublicDNS


— Create Spark Standalone Cluster
— create new namespace for spark cluster
kubectl create -f namespace-spark-cluster.yaml
— set up some environment variables
./environmentSetUp.sh
— start the master, service and worker
kubectl apply -f spark-cluster.yaml
— check if all the container images are pulled and deployed
kubectl get all

— open spark-shell now
kubectl exec masterNode -it spark-shell
— run few queries
sc.hadoopConfiguration.set("fs.s3a.awsAccessKeyId", "ProvideYourKeyIDValue")
sc.hadoopConfiguration.set("fs.s3a.awsSecretAccessKey","ProvideYourAccessKey")
-- upload sample data file in this example it is testclients.csv
val clients = sc.textFile("s3a://kubernetes-aws-test-data/testclients.csv")
clients.count()
val output = clients.filter(line => line.contains("(973)"))
output.saveAsTextFile("s3a://kubernetes-aws-test-data/output1")

—- submit spart job in to Kubernetes cluster directly in cluster mode. Specify the master URL. To get the master URL run "kubectl config view". In -- the below example, you are submitting the job in default name space of Kubernetes. Kubernetes will create a pod with spark driver and executer -- images mentioned in the command. The spark cluster runs on the pod created by kubernetes.

./spark-submit --deploy-mode cluster   --class org.apache.spark.examples.SparkPi   --master k8s://https://api-myfirstcluster-k8s-lo-hqulii-1576808915.us-west-2.elb.amazonaws.com   --kubernetes-namespace default   --conf spark.executor.instances=5   --conf spark.app.name=spark-pi   --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.5.0   --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.5.0   local:///opt/spark/examples/jars/spark-examples_2.11-2.2.0-k8s-0.5.0.jar


—- dashboard addon. Install create heapster pod and dashboard pod. Heasper is required if you want CPU and memory usage in the dashboard.
kubectl apply -f heapster.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

-- to access dashboard, get the token by running the below command. Use token for kubernetes.io/service-account-token
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
-- create a proxy to access the dashboard from localhost
kubectl proxy
—- access dashborad
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

—-send the logs to elk. Run the below commands
cp ~/.kube/config .
chmod 644 config
kubectl --kubeconfig=~/.kube/config apply -f k8-es-simple-service.yml
kubectl --kubeconfig=./config apply -f k8-es-simple-service.yml
kubectl --kubeconfig=./config apply -f k8-fluentd-ds.yml
kubectl --kubeconfig=./config apply -f k8-kibana-service.yml

-- To access Kibana dashboard from localhost, create a proxy
kubectl --kubeconfig=./config proxy
-- access Kinana from the below UI
http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kibana-logging

—- send the logs to s3
-- update the s3_iam_role_complete.json to suit your needs
aws iam create-policy --policy-name kubernetes-fluentd-s3-logging --policy-document file://s3_iam_role_complete.json
-- create a role. Role name can be anything of your choice
aws iam attach-role-policy --policy-arn arn:aws:iam::657115194624:policy/kubernetes-fluentd-s3-logging-3 --role-name theroleyoucreated
-- run the below command to set the configmap
kubectl create configmap fluentd-conf --from-literal=AWS_S3_BUCKET_NAME=kubernetes-aws-test-logs \
    --from-literal=AWS_S3_LOGS_BUCKET_PREFIX=logs  \
    --from-literal=AWS_S3_LOGS_BUCKET_PREFIX_KUBESYSTEM=system \
    --from-literal=AWS_S3_LOGS_BUCKET_REGION=us-east-1 \
    --from-file=fluent_s3.conf -n kube-system

kubectl create -f fluentd_s3.yaml
--view the logs --
aws s3api list-objects --bucket nameofthebucketforlogs
 
—- delete cluster  --
kops delete cluster --name myfirstcluster.k8s.local --state s3://kubernetes-aws-dakshin --yes


