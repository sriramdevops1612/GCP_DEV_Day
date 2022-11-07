### Pre req
https://www.cloudskillsboost.google/focuses/1029?parent=catalog

https://www.cloudskillsboost.google/focuses/564?parent=catalog



### GCP Authorization
```
gcloud auth list

gcloud config list project
```

### set the zone
```
gcloud config set compute/zone us-east1-b

```

### get sample code

```
gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .

cd orchestrate-with-kubernetes/kubernetes

```

### create a cluster with 3 nodes
```
gcloud container clusters create bootcamp \
  --machine-type e2-small \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```
### learn about deployment

```
kubectl explain deployment

kubectl explain deployment --recursive

kubectl explain deployment.metadata.name

```

### Create a deployment
```
vi deployments/auth.yaml

i

### change the image
...
containers:
- name: auth
  image: "kelseyhightower/auth:1.0.0"
...

```
### deployment file after chnages
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:1.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
```
### create deployment
```
kubectl create -f deployments/auth.yaml

kubectl get deployments

 kubectl get replicasets
 kubectl get pods
 
 kubectl create -f services/auth.yaml
 

kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml


kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml

curl -ks https://<EXTERNAL-IP>

curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`
 
```

### Scale a deployment
```
kubectl explain deployment.spec.replicas

kubectl scale deployment hello --replicas=5

kubectl get pods | grep hello- | wc -l

kubectl scale deployment hello --replicas=3
kubectl get pods | grep hello- | wc -l
```
####Deployments support updating images to a new version through a rolling update mechanism. When a deployment is updated with a new version, it creates a new ReplicaSet and slowly increases the number of replicas in the new ReplicaSet as it decreases the replicas in the old ReplicaSet.

To update your deployment, run the following command:
```
kubectl edit deployment hello

...
containers:
  image: kelseyhightower/hello:2.0.0
...

kubectl get replicaset

kubectl rollout history deployment/hello

If you detect problems with a running rollout, pause it to stop the update.
kubectl rollout pause deployment/hello

Verify the current state of the rollout:
kubectl rollout status deployment/hello

you can also verify this on the Pods directly:
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

The rollout is paused which means that some pods are at the new version and some pods are at the older version.
We can continue the rollout using the resume command:
kubectl rollout resume deployment/hello

When the rollout is complete, you should see the following when running the status command:
kubectl rollout status deployment/hello


Rollback an update
Assume that a bug was detected in your new version. Since the new version is presumed to have problems, any users connected to the new Pods will experience those issues.
You will want to roll back to the previous version so you can investigate and then release a version that is fixed properly.

Use the rollout command to roll back to the previous version:
kubectl rollout undo deployment/hello

Verify the roll back in the history:
kubectl rollout history deployment/hello

Finally, verify that all the Pods have rolled back to their previous versions:
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'


Task 4. Canary deployments

When you want to test a new deployment in production with a subset of your users, use a canary deployment. Canary deployments allow you to release a change to a small subset of your users to mitigate risk associated with new releases.

Create a canary deployment
A canary deployment consists of a separate deployment with your new version and a service that targets both your normal, stable deployment as well as your canary deployment.

First, create a new canary deployment for the new version:
cat deployments/hello-canary.yaml

Now create the canary deployment:
kubectl create -f deployments/hello-canary.yaml

After the canary deployment is created, you should have two deployments, hello and hello-canary. Verify it with this kubectl command:
kubectl get deployments

On the hello service, the selector uses the app:hello selector which will match pods in both the prod deployment and canary deployment. However, because the canary deployment has a fewer number of pods, it will be visible to fewer users.

You can verify the hello version being served by the request:

curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

Run this several times and you should see that some of the requests are served by hello 1.0.0 and a small subset (1/4 = 25%) are served by 2.0.0.



Canary deployments in production - session affinity
In this lab, each request sent to the Nginx service had a chance to be served by the canary deployment. But what if you wanted to ensure that a user didn't get served by the Canary deployment? A use case could be that the UI for an application changed, and you don't want to confuse the user. In a case like this, you want the user to "stick" to one deployment or the other.

You can do this by creating a service with session affinity. This way the same user will always be served from the same version. In the example below the service is the same as before, but a new sessionAffinity field has been added, and set to ClientIP. All clients with the same IP address will have their requests sent to the same version of the hello application.

Due to it being difficult to set up an environment to test this, you don't need to here, but you may want to use sessionAffinity for canary deployments in production.

```
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

Task 5. Blue-green deployments
Rolling updates are ideal because they allow you to deploy an application slowly with minimal overhead, minimal performance impact, and minimal downtime. There are instances where it is beneficial to modify the load balancers to point to that new version only after it has been fully deployed. In this case, blue-green deployments are the way to go.

Kubernetes achieves this by creating two separate deployments; one for the old "blue" version and one for the new "green" version. Use your existing hello deployment for the "blue" version. The deployments will be accessed via a Service which will act as the router. Once the new "green" version is up and running, you'll switch over to using that version by updating the Service.


The service
Use the existing hello service, but update it so that it has a selector app:hello, version: 1.0.0. The selector will match the existing "blue" deployment. But it will not match the "green" deployment because it will use a different version.

First update the service:

kubectl apply -f services/hello-blue.yaml

Updating using Blue-Green deployment
In order to support a blue-green deployment style, we will create a new "green" deployment for our new version. The green deployment updates the version label and the image path.

Create the green deployment:
kubectl create -f deployments/hello-green.yaml

Once you have a green deployment and it has started up properly, verify that the current version of 1.0.0 is still being used:
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

Now, update the service to point to the new version:
kubectl apply -f services/hello-green.yaml

When the service is updated, the "green" deployment will be used immediately. You can now verify that the new version is always being used:
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version


Blue-Green rollback
If necessary, you can roll back to the old version in the same way.

While the "blue" deployment is still running, just update the service back to the old version:

kubectl apply -f services/hello-blue.yaml


Once you have updated the service, your rollback will have been successful. Again, verify that the right version is now being used:
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version










```

