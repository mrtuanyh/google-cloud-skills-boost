Secure Workloads in Google Kubernetes Engine: Challenge Lab.
 
## Task 1: Download the necessary files

Open Cloud Shell and run the following to collect all the files made available to you:

``` 
gsutil cp gs://spls/gsp335/gsp335.zip .
unzip gsp335.zip
```
 
### Create ENV VARs for Dynamic Variables:
``` 
DM_DEPLOYMENT=$(gcloud deployment-manager deployments list | grep NAME | cut -d ' ' -f2)
CLUSTER_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep security-demo | cut -d ' ' -f2)
CLOUD_SQL_INSTANCE_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep wordpress-db | cut -d ' ' -f2)
SA_NAME=$(gcloud deployment-manager deployments describe $DM_DEPLOYMENT | grep sa-wordpress | cut -d ' ' -f2)
```

## Task 2: Set up a cluster
To create a cluster for your demo, use the following values to ensure you have enough resources:
- Name: <Cluster Name>
- Zone: us-central1-c
- Machine-type: e2-medium
- Nodes: 2
- Enable network policy

``` 
gcloud container clusters create $CLUSTER_NAME  --machine-type n1-standard-4 --num-nodes 2 --zone us-central1-c --enable-network-policy
gcloud container clusters get-credentials $CLUSTER_NAME --zone us-central1-c
```

## Task 3: Set up WordPress

Set up the Cloud SQL database, database username and password

1. Create a Cloud SQL instance called Cloud SQL Instance in us-central1, using the default values.
2. Create a Cloud SQL database for WordPress.
3. Create a user using the following values:
- Username: wordpress
- Access from host %
- Access to the Cloud SQL instance you created
- Set a password (remember it, you will need later)

``` 
gcloud sql instances create $CLOUD_SQL_INSTANCE_NAME --region=us-central1
gcloud sql databases create wordpress --instance $CLOUD_SQL_INSTANCE_NAME
gcloud sql users create wordpress --instance=$CLOUD_SQL_INSTANCE_NAME --host=% --password='P@ssword!'
```

Create a service account for access to your WordPress database from your WordPress instances

1. Create a service account called Service Account .
2. Bind the service account to your project, give the role roles/cloudsql.client.
3. Save the service account credentials in a json file.
4. Save the service account json file as a secret in your Kubernetes cluster.

Use this as a template: 

``` 
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME
 
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT    --role roles/cloudsql.client  --member serviceAccount:$SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
 
gcloud iam service-accounts keys create key.json     --iam-account $SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

5. Save the WordPress database username and password you used in step #4 as secrets in your Kubernetes cluster.

Use this as a template:

```
kubectl create secret generic cloudsql-instance-credentials     --from-file key.json
kubectl create secret generic cloudsql-db-credentials \
 --from-literal username=wordpress \
 --from-literal password='P@ssword!'
```

Create the WordPress deployment and service
1. Use kubectl create -f volume.yaml to create a persistent volume for your WordPress application.
2. Open wordpress.yaml and replace INSTANCE_CONNECTION_NAME with the instance name of your Cloud SQL database (the format is project:region:databasename).
3. Use kubectl apply -f wordpress.yaml to create the WordPress environment.

``` 
kubectl apply -f volume.yaml
 
sed -i s/INSTANCE_CONNECTION_NAME/${GOOGLE_CLOUD_PROJECT}:us-central1:$CLOUD_SQL_INSTANCE_NAME/g wordpress.yaml
 
kubectl apply -f wordpress.yaml
```
 
## Task 4: Setup ingress with TLS

Set up ingress-nginx environment

Using Helm, install the latest NGINX Ingress controller (ingress-nginx) in the default namespace.

Call it ingress-nginx.

``` 
helm upgrade --install ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx
 
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```

Set up your DNS record

Check the service ingress-nginx-controller as an external IP address before continuing (kubectl get svc) to the next step.

```
kubectl get svc
 
sed -i s/LAB_EMAIL_ADDRESS/$SA_NAME@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com/g issuer.yaml
 
kubectl apply -f issuer.yaml
```

If you face any internal error, execute the command again

``` 
. add_ip.sh
```
You should see record added. You now have a DNS record added of the format YOUR_LAB_USERNAME.labdns.xyz which points to your ingress cluster (the script will output your DNS name).

Set up cert-manager.io

Install cert-manager.io (make sure you use the right version - v1.8.0 works well with this lab).

Edit issuer.yaml and set the email address to Service Account @ Project ID .iam.gserviceaccount.com.

Use kubectl apply -f issuer.yaml to setup the letsencrypt prod issuer.

```
HOST_NAME=$(echo $USER | tr -d '_').labdns.xyz
sed -i s/HOST_NAME/${HOST_NAME}/g ingress.yaml
 
kubectl apply -f ingress.yaml
```
 
## Task 5: Set up a Network Policy
Show your colleagues how to restrict the network traffic using the provided network-policy.yaml file. It's not complete; you will need to add the configuration to allow ingress traffic from the world into ingress-nginx, but you can solve that easily.

``` 
kubectl apply -f network-policy.yaml
 
nano network-policy.yaml
```

``` 
--------- Add the following code at the last line-------

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: allow-nginx-access-to-internet
spec:
 podSelector:
  matchLabels:
   app.kubernetes.io/name: ingress-nginx
 policyTypes:
 - Ingress
 ingress:
 - {}
 
=================================================================
kubectl apply -f network-policy.yaml
 
=================================================================

``` 
gcloud services enable \
 container.googleapis.com \
 containeranalysis.googleapis.com \
 binaryauthorization.googleapis.com
 
gcloud container clusters update $CLUSTER_NAME --enable-binauthz --zone us-central1-c
```
 
Task 7:

``` 
gcloud container binauthz policy export > bin-auth-policy.yaml
 
nano bin-auth-policy.yaml
``` 

``` 
#### Edit and add the four values and change
admissionWhitelistPatterns:
- namePattern: docker.io/library/wordpress:latest
- namePattern: us.gcr.io/k8s-artifacts-prod/ingress-nginx/*
- namePattern: gcr.io/cloudsql-docker/*
- namePattern: quay.io/jetstack/*
defaultAdmissionRule:
 enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
 evaluationMode: ALWAYS_DENY
globalPolicyEvaluationMode: ENABLE
name: projects/<Project_ID>/policy
##########
```

``` 
gcloud container binauthz policy import bin-auth-policy.yaml
```

Go to Security > Binary Authorization, click EDIT POLICY. Change Default rule to Disallow all images and SAVE POLICY.
 
Edit psp-restrictive.yaml

``` 
nano psp-restrictive.yaml
```

Change policy/v1 to policy/v1beta1

``` 
kubectl apply -f psp-restrictive.yaml
kubectl apply -f psp-role.yaml
kubectl apply -f psp-use.yaml
```