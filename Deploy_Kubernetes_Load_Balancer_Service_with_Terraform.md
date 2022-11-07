
```
gsutil -m cp -r gs://spls/gsp233/* .

cd tf-gke-k8s-service-lb

cat main.tf

cat k8s.tf

terraform init

terraform apply


```
In the console, navigate to Navigation menu > Kubernetes Engine.

Click on tf-gke-k8s cluster and check its configuration.

In the left panel, click Services & Ingress and check the nginx service status.

Click the Endpoints IP address to open the Welcome to nginx! page in a new browser tab.
