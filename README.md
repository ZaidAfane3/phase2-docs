# Documentation for Elm Phase 2 Technical Assesment

## Repos: 
- infra (Need Access Request): [https://github.com/ZaidAfane3/todo-infra](https://github.com/ZaidAfane3/todo-infra)
- todo-be: [https://github.com/ZaidAfane3/todo-be](https://github.com/ZaidAfane3/todo-be)
- todo-auth: [https://github.com/ZaidAfane3/todo-auth](https://github.com/ZaidAfane3/todo-auth)
- todo-fe: [https://github.com/ZaidAfane3/todo-fe](https://github.com/ZaidAfane3/todo-fe)


# Infra Setup :
The infrastructure is built with terraform in a modular using Google Cloud Servie (GCS) buckets. 

A bucket was created for the terrafrom statefile and rhw following resources are created. 

## vpc:
This module creates the network to run the k8s cluster in along with it's subnets and the main firewall rules.

- `k8s-network`:
    - 2 public subnets (Exposed to the Internet)
    - 2 private subnets (Accessed through bastion or load balancer)
- Firewall Rules: 
    - Allow k8s api server port from private cidr ips 
    - Allow the node port range
    - Allow Calico port
    - Allow Calico Protocl
    - (For Testing) Allow tcp/udp/icmp on private cidr ips
    - Allow Kublet port
    - Allow SSH from bastion to private subnets

## k8s-unmanaged
The module implements the computational resources and the permissions for the service account that will be used by the k8s cluster.

All k8s nodes are network tagged for load balancer backend.

The permissions assigned to the service account are:
    - VM instance admin
    - artifact registry read (docker image)
    - storage/disks admin
    - compute service read
    - impersonation permission
    - compute service admin

The controlplane node was configured with the following start-up script and at the end created a kubedm join token with no expiration to be used for worker node 

```sh
#!/bin/bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

## Install the necessary packages
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gpg \
    lsb-release

### install k8s
## Add the Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

## Check and Install Latest Version of Kubernetes is 1.34.1-1.1
apt-cache policy kubelet kubeadm kubectl

sudo apt-get install -y \
    kubelet=1.34.1-1.1 \
    kubeadm=1.34.1-1.1 \
    kubectl=1.34.1-1.1

# Hold the verions of these packages so when apt upgrade/update is run, it doesn't upgrade these packages
sudo apt-mark hold kubelet kubeadm kubectl

# Verify Installation 
apt list | grep -E  "kubeadm|kubectl|kubelet" | grep -i installed

kubectl version --client
kubeadm version
kubelet --version


## setup container runtime before installing k8s
sudo apt update
sudo apt install -y containerd
cat /usr/lib/systemd/system/containerd.service 
sudo ls /usr/sbin/runc
sudo ls /opt/cni/bin

containerd --version

# Match cgroup driver for containerd with kublet driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd

## Configure Netowrking
# Enable IP Forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-kubernetes-ipforwarding.conf


cat /etc/sysctl.d/99-kubernetes-ipforwarding.conf

sudo sysctl --system
sudo sysctl net.ipv4.ip_forward
```

and manually ran the following commands to starts the k8s cluster:
```sh 
#!/bin/bash
sudo kubeadm init
sudo kubeadm token create --ttl 0 --print-join-command 

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "alias k=kubectl" >> ~/.bashrc
source ~/.bashrc

kubectl cluster-info
```
## worker-nodes
The module creates a scalable instance group managr with auto scaling. 

The worker nodes will be created from a an instance template that uses a custom built image, with the following configuraton, and the start-up script in terraform will run the kubeadm join:  
```shell
#!/bin/bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

## Install the necessary packages
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gpg \
    lsb-release

### install k8s
## Add the Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

## Check and Install Latest Version of Kubernetes is 1.34.1-1.1
apt-cache policy kubelet kubeadm kubectl

sudo apt-get install -y \
    kubelet=1.34.1-1.1 \
    kubeadm=1.34.1-1.1 \
    kubectl=1.34.1-1.1

# Hold the verions of these packages so when apt upgrade/update is run, it doesn't upgrade these packages
sudo apt-mark hold kubelet kubeadm kubectl

# Verify Installation 
apt list | grep -E  "kubeadm|kubectl|kubelet" | grep -i installed

kubectl version --client
kubeadm version
kubelet --version


## setup container runtime before installing k8s
sudo apt update
sudo apt install -y containerd
cat /usr/lib/systemd/system/containerd.service 
sudo ls /usr/sbin/runc
sudo ls /opt/cni/bin

containerd --version

# Match cgroup driver for containerd with kublet driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd

## Configure Netowrking
# Enable IP Forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-kubernetes-ipforwarding.conf


cat /etc/sysctl.d/99-kubernetes-ipforwarding.conf

sudo sysctl --system
sudo sysctl net.ipv4.ip_forward
```

## bastion-host
The module create a bastion host with static ip configured to be accessed through ssh with fail2ban and restricted ssh access. 



# Kubernetes Cluster
The service account wuith all the permissions mentioned before is attached to all k8s nodes (controlplane and workers)

## Cluster Networking
Once the cluster is started, the node are not ready because the Container Network Interface (CNI) is not deployed by default so Calico was installed to start network communication inside the cluster. 
```sh 
k apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

## Storage Provisioning
To enable the workload to use privisoned storage on gcp through pv/pvc we need a Container Storage Interface (CSI) to be able to communicate with underlaying gcp apis. 

- Github Repo: [https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver](https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver)

```sh
kubectl create namespace gce-pd-csi-driver

kubectl delete -k https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver/deploy/kubernetes/overlays/stable-master
```

after applying the csi driver there were a couple of issues and I had to edit the controller deployment/daemonset with the following: 
1. Add tolerations for controlplane node
2. remove `--enable-multitenancy` from deployment and daemon-set because it was causing an issue because it has to do smth with GKE. 

Finally, the Storage Class is deployed using the manifest found [`k8s/cluster/storageclass.yaml`](https://github.com/ZaidAfane3/todo-infra/blob/main/k8s/cluster/storageclass.yaml).

## Image Pull Secret
To create the secret a key in json format is needed to authenticate conatinerd with gcp artifcate registry

```sh
gcloud iam service-accounts keys create key.json \
    --iam-account=k8s-nodes-svc-account@personal-473312.iam.gserviceaccount.com
```

the pull secret is bound by namespace, so to make life easier `kubernetes-replicator` is deployed to replicate the secret to all namespace
```sh
kubectl apply -f https://raw.githubusercontent.com/mittwald/kubernetes-replicator/master/deploy/rbac.yaml

kubectl apply -f https://raw.githubusercontent.com/mittwald/kubernetes-replicator/master/deploy/deployment.yaml
```

and then the secret applied 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gcp-registry-secret
  namespace: default
  annotations:
    replicator.v1.mittwald.de/replicate-to: "*"
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: JSON_KEY_B64 
```

## Metrics Server
Deployed using the manifest found [`k8s/cluster/metrics-server.yaml`](https://github.com/ZaidAfane3/todo-infra/blob/main/k8s/cluster/metrics-server.yaml).


## Nginx Ingress
nginx ingress controller when setup it created a loadbalancer service resources but in unamnaged k8s loadbalancer service act as nodeport

used the script to deploy nginx ingress controller
```sh
#!/bin/bash

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.kind=DaemonSet \
  --set controller.hostNetwork=true \
  --set controller.hostPort.enabled=true \
  --set controller.hostPort.ports.http=80 \
  --set controller.hostPort.ports.https=443 \
  --set controller.service.create=false \
  --set controller.allowSnippetAnnotations=true \
  --set controller.admissionWebhooks.enabled=false

# Eliminate Any Validation Issues
kubectl delete validatingwebhookconfigurations nginx-ingress-ingress-nginx-admission

# Get the NodePort 
kubectl get svc -n ingress-nginx
```

Now a cloud loadbalancer pointing to the node port is needed to access webapps deployed on the cluster, and the steps are as follow: 
### Create an SSL cert: 
```sh 
#!/bin/bash 

certbot certonly \
  --manual \
  --preferred-challenges=dns \
  --email afaneh.zaid98@gmail.com \
  --server https://acme-v02.api.letsencrypt.org/directory \
  -d gcp.afaneh.xyz \
  -d '*.gcp.afaneh.xyz'


# Certificate is saved at: /etc/letsencrypt/live/gcp.afaneh.xyz/fullchain.pem
# Key is saved at:         /etc/letsencrypt/live/gcp.afaneh.xyz/privkey.pem
```  

### Create ALB on GCP Console: 
1. Choose HTTP/HTTPS Application LoadBalancer
2. Public Facing
3. Global Region
4. Configure Frontends (created a static ip to be used for both)
    - http:  listening on 80 
    - https: listening on 443 with the cert generated by certbot (Let's Encrypt)
5. Create a backend pointing to the instance group created in the terraform
6. Relay on Ingress to redirect http to https

for the ALB health check, a health check app was deployed using the manifest [`k8s/cluster/health-check.yaml`](https://github.com/ZaidAfane3/todo-infra/blob/main/k8s/cluster/health-check.yaml)

## Monitoring 
Deployed using kube-promethues-stack helm 

- [Helm Chart on ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)
- [GitHub Repository](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

```sh
#!/bin/bash

kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set alertmanager.enabled=false \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=standard-rwo \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=5Gi \
  --set prometheus.prometheusSpec.retention=3d \
  --set prometheus.prometheusSpec.resources.requests.memory=512Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=standard-rwo \
  --set grafana.persistence.size=5Gi
```

Then exposed Grafana through the ingress defined in [`infra/k8s/cluster/monitoring/ingress.yaml`](https://github.com/ZaidAfane3/todo-infra/blob/main/k8s/cluster/monitoring/ingress.yaml).

## ArgoCD 
Deployed using the script 
```sh
#!/bin/bash

kubectl create namespace argocd 

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get the password for the admin
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Then expose argocd through the ingress in `infra-repo/k8s/cluster/argocd/ingress.yaml`.

For argocd to access the infra gitops repo, ssh-key authentication was used and added ti argocd ui, then pointed each application to the path inside the infra repo. 


# Microservice Architecture
The to-do app web app is a simple 4 services
1. auth service: has it's own database and responsible for logging in/out and check if user is logged in through a cookie.
2. be service: managing the logig for a todo app and communicates with an OpenAI Compatible API for some fancy LLM integration, it talk also with the auth service through a ClusterIP service to check if user is authenticated, be has it's own database
3. frontend service: pointing to /api/todo and /auth 
4. The postgresql running with be and auth schema. 

## Postgresql 
Before Deploying the todo app 
the schemas for the services must be created in the database
```sh 
# Create Schemas
kubectl exec postgres-0 -n microservices -- bash -c 'export PGPASSWORD=$POSTGRESQL_PASSWORD && psql -U "$POSTGRESQL_USERNAME" -c "CREATE DATABASE auth_db;"'

kubectl exec postgres-0 -n microservices -- bash -c 'export PGPASSWORD=$POSTGRESQL_PASSWORD && psql -U "$POSTGRESQL_USERNAME" -c "CREATE DATABASE todo_db;"' 

# Grant Priviliges
kubectl exec postgres-0 -n microservices -- bash -c 'export PGPASSWORD=$POSTGRESQL_PASSWORD && psql -U "$POSTGRESQL_USERNAME" -c "GRANT ALL PRIVILEGES ON DATABASE auth_db TO $POSTGRESQL_USERNAME;"'

kubectl exec postgres-0 -n microservices -- bash -c 'export PGPASSWORD=$POSTGRESQL_PASSWORD && psql -U "$POSTGRESQL_USERNAME" -c "GRANT ALL PRIVILEGES ON DATABASE todo_db TO $POSTGRESQL_USERNAME;"'

# List Database
kubectl exec postgres-0 -n microservices -- bash -c 'export PGPASSWORD=$POSTGRESQL_PASSWORD && psql -U "$POSTGRESQL_USERNAME" -c "\l"'
```

## GitOps
All todo services has their own repos with docker images built and pushed to artifcate registry using github action workflows. 

The deployment files using kustomize are found for each service under the infra repo at [`8s/todo`](https://github.com/ZaidAfane3/todo-infra/tree/main/k8s/todo)

### CI
The CI will build the image and scan it with trivy when a new code is pushed to dev

### CD
CD will run on two stages `build-and-push` and `deploy`. 

On `build-and-push` the image is being built, tagged and pushed to the registry 
On `deploy` the gitops repo is checkout out and the deployment file is updated with the new tag and the changes are commited for argocd to pick it up and deploy it.

## Horizontal Pod Autoscale (HPA) 
Each one of the 3 services is configured with HPA in the same gitops repo. 


# Manual
These resources were created mnaually 
- Firewalls: 
    - Allow alb health checks
    - Allow http/https to the loadbalancer
- Loadbalancer
- Artifcat Registry

# How To 
## Generate Service Account Json Creds for Github Action
```
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions Artifact Registry" \
  --project=personal-473312

gcloud projects add-iam-policy-binding personal-473312 \
  --member="serviceAccount:github-actions-sa@personal-473312.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud iam service-accounts keys create github-actions-key.json \
  --iam-account=github-actions-sa@personal-473312.iam.gserviceaccount.com

jq -c . github-actions-key.json > github-actions-key.json
```
## Access the Node (Bypass Bastion)
```
### Access the Node
cat ~/.ssh/keys/gcp-perosnal/google_compute_engine.pub | pbcopy

## Whitelist my public IP address on the firewall
MY_PUBLIC_IP=$(curl -s https://api.ipify.org)
gcloud compute firewall-rules delete allow-ssh-from-zaid --quiet || true
gcloud compute firewall-rules create allow-ssh-from-zaid \
    --network=k8s-network \
    --allow=tcp:22 \
    --source-ranges=$MY_PUBLIC_IP/32

## SSH into the node
IP_ADDRESS=$(gcloud compute instances describe controlplane-node --zone=us-central1-a --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
ssh -i ~/.ssh/keys/gcp-perosnal/google_compute_engine zafaneh@$IP_ADDRESS -v
```


> Note: AI will be connecting to Ollam Running on my laptop through a cloudflared tunnel, running on vm is expensive.  