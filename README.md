# case-study-elastic
- Create a new storage class appropriate for ElasticSearch, set it as default, retain data on the PV in case of PVC deletion
- Securely store credentials for ElasticSearch (follow best practices)
- Set up ElasticSearch in a high-availability configuration (3-node cluster):
- Ensure that ElasticSearch nodes are placed on dedicated nodes ("ElasticSearch only") and do not run on the same node
- Expose the ElasticSearch service to the internet
- Implement a certificate management solution to automatically issue and renew SSL certificates for the ElasticSearch service. Ensure that the Elastic exposes SSL/TLS certificates.


Workaround:
- install helm, docker, k3d, kubectl on my virtual private server
- create k3d cluster (3 nodes + 1 control plane master node)
- deploy nginx:
  - helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  - helm repo update
  - kubectl create namespace ingress-nginx --dry-run=client -o yaml | kubectl apply -f -
  - helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx
