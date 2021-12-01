**Task:**

Install Kubernetes cluster (anywhere you like) and deploy WordPress and Istio. Expose
service to be accessible via Browser

**Environment used:**

- Google Kubernetes Engine (GKE) cluster

**Project Creation in GCP:**

- Create a new project, I have created a project as below

  - Project Name - My First Project
  - Project ID - eternal-mark-328015
  - WORKING\_DIR=$(pwd)

**Applying IAM Conditions and enabling API:**

- In GCP, go to IAM page and click add
- In the New Principals input box, enter the service account email.
- Click the Role dropdown list and select the Cloud SQL Client role
- Click Add condition and enter a title
- Select the Condition Editor tab.
- In the Condition Builder section, add 3 conditions:
  - For Condition type - Resource - Type, select sqladmin.googleapis.com/Instance
  - For Condition type - Resource - Name, enter eternal-mark-328015
  - For Condition type - Resource - Service, select sqladmin.googleapis.com
- Save the condition and save the policy
- Come back to the cloud shell, enable the GKE and Cloud SQL admin API

**Setting up GCP environment:**

- In Cloud Shell, set the default zone
  - gcloud config set compute/zone us-west4-b
- Set the PROJECT\_ID
  - export PROJECT\_ID=eternal-mark-328015

**Creating GKE cluster:**

- Create a new cluster named persistent-disk-tutorial with 3 nodes:
  - CLUSTER\_NAME=persistent-disk-tutorial
 gcloud container clusters create $CLUSTER\_NAME \
     --num-nodes=3 --enable-autoupgrade --no-enable-basic-auth \
     --no-issue-client-certificate --enable-ip-alias --metadata \
     disable-legacy-endpoints=true

**Creating PVC for wordpress:**

- I am creating PVC with storage class name as wordpress-volumeclaim.yaml
- Now deploy this file (wordpress-volumeclaim.yaml)
  - Kubectl apply –f wordpress-volumeclaim.yaml
- Check for the status of PVC, using
  - kubectl get pvc

**Creating cloud SQL for MySQL instance:**

- Creating a instance named mysql-wordpress-instance
  - INSTANCE\_NAME=mysql-wordpress-instance
 gcloud sql instances create $INSTANCE\_NAME
- Add instance connection name as environment variable
  - export INSTANCE\_CONNECTION\_NAME=$(gcloud sql instances describe $INSTANCE\_NAME \
     --format=&#39;value(connectionName)&#39;)
- Create a database for wordpress to store its data
  - gcloud sql databases create wordpress --instance $INSTANCE\_NAME
- Create a database user called wordpress and a password for WordPress to authenticate to the instance
  - CLOUD\_SQL\_PASSWORD=$(openssl rand -base64 18)
 gcloud sql users create wordpress --host=% --instance $INSTANCE\_NAME \
     --password $CLOUD\_SQL\_PASSWORD

**Configuring service account and create secrets:**

- To let your WordPress app access the MySQL instance through a Cloud SQL proxy, create a service account
  - SA\_NAME=cloudsql-proxy
 gcloud iam service-accounts create $SA\_NAME --display-name $SA\_NAME
- Add the service account email address as an environment variable
  - SA\_EMAIL=$(gcloud iam service-accounts list \
     --filter=displayName:$SA\_NAME \
     --format=&#39;value(email)&#39;)
- Add the cloudsql.client role to your service account
  - gcloud iam service-accounts keys create $WORKING\_DIR/key.json \
     --iam-account $SA\_EMAIL
- Create a Kubernetes secret for the MySQL credentials
  - kubectl create secret generic cloudsql-db-credentials \
     --from-literal username=wordpress \
     --from-literal password=$CLOUD\_SQL\_PASSWORD
- Create a Kubernetes secret for the service account credentials
  - kubectl create secret generic cloudsql-instance-credentials \
     --from-file $WORKING\_DIR/key.json

**Deploying Wordpress:**

- Deploy the file (wordpress\_cloudsql.yaml)
  - Kubectl create –f wordpress\_cloudsql.yaml
- Watch the deployment using
  - Kubectl get pods –watch
- After few mins the pods starts to run

**Expose the wordpress service:**

- Create the deployment (wordpress-service.yaml) and watch it by using
  - Kubectl create –f wordpress-service.yaml
  - Kubectl get svc –watch
- It will create a svc and will show a external IP, using which we can access wordpress in the browser

**Downloading and Installing Istio:**

Execute the below commands to download and install istio

- To download the package
  - curl -L https://istio.io/downloadIstio | sh -
- Go to Istio package directory
  - cd istio-1.12.0
- Add the istioctl client to your path
  - export PATH=$PWD/bin:$PATH
- Install Istio
  - istioctl install --set profile=demo –y

**Routing traffic through Istio to access Wordpress application:**

- Now we can see there is a separate namespace created for istio - &#39;**istio-system**&#39;

- If you try to access the newly created WordPress using the external IP, in a browser, you will notice that nothing is served there. This is because Istio blocks all traffic coming to the service mesh which is not coming through one of its envoy proxies, and to enable it we have a gateway configuration (wordpress-gateway.yaml)
  - kubectl create -f wordpress-gateway.yaml
- And now we have the gateway created but there is no route. We have to create a route that will forward the traffic coming through the gateway to appropriate service (virtual-route.yaml)
  - kubectl create -f virtual-route.yaml
- This will route the traffic that is getting in through the wordpress-gateway on port 80
- I am now editing the svc of wordpress and changing the service type from LoadBalancer to ClusterIP, so that we cannot access the wordpress application directly
  - kubectl edit svc wordpress
- Lets now check the external ip of &#39;istio-ingressgateway&#39;
  - Kubectl get svc –n istio-system

