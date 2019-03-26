# redis-gke-bootstrap
Get redis running on GKE with a demo app. For Redis Conf 2019

# Pre-work
If you have never used Google Cloud Platform before, create a new account here: https://cloud.google.com/free/

To get started, click the following button:

[![Open in Cloud Shell](//gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/thesandlord/redis-gke-bootstrap.git)

This will open Google Cloud Shell and automatically clone this repository.

# Create your cluster

First, let's create a small GKE cluster that you can use to run Redis and some sample apps in.

We will use the gcloud cli to provision the cluster:

```
gcloud container clusters create "redis-cluster" --zone "us-west2-b" --machine-type "g1-small" --num-nodes "2"
```

## Warning: You will be charged for this cluster! Make sure you delete the cluster (see last step) once you are done, otherwise it will consume all of your free credits.

This will create a small 2-node cluster in your current GCP project.


# Deploying redis

Next, let's deploy Redis to this clusters

```
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml

kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml
```

Great! This will deploy a single master redis instance, and create a endpoint that you can connect to.

Let's try it out!

# Connecting to Redis locally

Connect to the Redis master and run the redis-cli

```
kubectl exec -it $(kubectl get pods -l app=redis -o jsonpath={.items[0].metadata.name}) -- redis-cli
```

This allows us to connect to the redis master without messing with firewalls or creating load balancers!

You can run this command anytime to connect to your database.

Type `exit` to close the connection.

# Deploy a sample app that connects to your Redis instance

During the course of the conference, speakers will share their code as a Docker container you can run in your cluster. Let's see a sample of that in action.

## Deploy the App

Let's deploy a sample app that will connect to Redis.

The container is called "gcr.io/google-samples/gb-frontend:v4" and is a simple PHP guestbook that uses Redis to store information.

To run this container, run:

```
kubectl run php-guestbook --image=gcr.io/google-samples/gb-frontend:v4
```

## Connect to the App

To save time and costs, we are going to use a NodePort to connect to the service.

First, create a Firewall Rule that will expose NodePorts to the internet.

```
gcloud compute firewall-rules create gke-node-ports --allow tcp:30000-32767
```

Then, find the IP addresses of your node and save it to an environment variable:
```
NODE=$(kubectl get nodes -o jsonpath='{ $.items[0].status.addresses[?(@.type=="ExternalIP")].address }')
```


To connect to this container, we need to create a service:

```
kubectl expose deployment php-guestbook --type=NodePort --port=80 --target-port=80
```

This will expose a port on the IP address you found earlier. Save the port to an environment variable:

```
PHP_GUESTBOOK_PORT=$(kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services php-guestbook)
```

Remember, each service you expose with a NodePort will have a different port number, so make sure you save them to different variables.

Now, you can visit the combined URL to see your new app in action:

```
echo http://${NODE}:${PHP_GUESTBOOK_PORT}
```

# Cleaning Up

To clean everything up once you are done, delete the Firewall Rule and the Cluster

```
gcloud compute firewall-rules delete gke-node-ports
gcloud container clusters delete "redis-cluster" --zone "us-west2-b"
```

# Advanced

This guide is designed to get you started with Kubernetes, but there is a lot more to learn!

If you need more advanced Kubernetes help, or are confused at any time, please drop by the Google Cloud Booth!