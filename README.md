# Create the perfect shared hosting on Kubernetes

## Baby steps to greatness

### 1. NFS server
Google Cloud (GCP) does not allows multiple server to mount a single disk in _ReadWrite_ mode. We need to create an NFS server to share a single Persistent Disk (PD) across multiple pods.

First we need to create our PD. We could use the built-in Kubernetes StorageClass `standard` to create our PD. But doing so we would lose the ability to name the disk. Naming the disk allows us to write snapshot automation. In case of emergency, we could recreate the disk from a snapshot with the same name and the cluster would mount it back.

`gcloud compute --project "your-project" disks create "the-disk-name" --size "the-disk-size-in-gigabytes" --zone "us-central1-a" --type "pd-standard"`

Second, we need to replace `pdName: the-disk-name` with the name you specified in previous command in the `nfs.yaml` file.

`kubectl create -f deploy/nfs.yaml`

### 2. HTTP/S proxy server
Kubernetes default Ingress controller uses Google Cloud Load Balancer (LB). Each Ingress rules we will specify (one for each site) will create a forward rules on the LB. Each forward rules cost $16.80/month. We could easily host 300 sites on the cluster, which would cost $2 066.40/month for the forwarding rules alone. We dont want that !

So we'll want to create our own proxy for the forwarding rules. We'll use Traefik which has built-in support for Kubernetes Ingress.

Based on [this](https://blog.osones.com/en/kubernetes-traefik-and-lets-encrypt-at-scale.html), in the future we might be able to use [Kubernetes' builtin etcd](https://github.com/containous/traefik/issues/926).

`kubectl create -f deploy/traefik.1.yaml`

Because of this [bug](https://github.com/containous/traefik/issues/927), we need to manualy delete a key from Consul.

`kubectl --namespace=your-namespace exec -it traefik-consul-0 consul kv delete traefik/acme/storagefile`

Only then we can start up Traefik

`kubectl create -f deploy/traefik.2.yaml`

Note the external IP of the traefik service. This IP will be used for your DNS. Might also be a good idea to reserve that IP.

`kubectl --namespace=your-namespace get service --selector="app=traefik"`

### 3. First pod
See `hello-perl.yaml`