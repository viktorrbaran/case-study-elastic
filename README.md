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
- deploy cert-manager:
  - helm repo add jetstack https://charts.jetstack.io
  - helm repo update
  - kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl apply -f -
  - helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.14.4 --set installCRDs=true
- deploy: Lets Encrypt Cluster Issuer - apply following manifest:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: vudcaa@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```
- create Retain StorageClass and set as default (Retain SC = if I remove PVC, PV will not be removed - no data loss)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: false
```
  - Currently, both are set as default, so we will mark the original one as not default:
    kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}' ()
- deploy ECK operator:
  - helm repo add elastic https://helm.elastic.co
  - helm repo update
  - kubectl create namespace elastic-system --dry-run=client -o yaml | kubectl apply -f -
  - helm install eck-operator elastic/eck-operator --namespace elastic-system
- set label + taint 3 worker nodes as ES only
  - for n in k3d-es-lab-agent-0 k3d-es-lab-agent-1 k3d-es-lab-agent-2; do kubectl label node $n node-role=elasticsearch --overwrite kubectl taint node $n elasticsearch=true:NoSchedule --overwrite done
- deploy elastic search:
  - kubectl create namespace elasticsearch
  - apply following manifest:
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es
  namespace: elasticsearch
spec:
  version: 8.12.2
  nodeSets:
  - name: nodes
    count: 3
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        nodeSelector:
          node-role: elasticsearch
        tolerations:
        - key: "elasticsearch"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: es
              topologyKey: kubernetes.io/hostname
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: "-Xms1g -Xmx1g"
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              memory: "2Gi"
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path-retain
        resources:
          requests:
            storage: 10Gi
```
- check credentials for elastic user (ECK created secret automatically)
  - kubectl -n elasticsearch get secret es-es-elastic-user -o jsonpath='{.data.elastic}' | base64 --decode; echo
- Expose ES to Internet - create ingress with cert-manager annotation (cert-manager will create certificate and TLC secret)
  - apply following manifest:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: es
  namespace: elasticsearch
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - ${ES_HOST}
    secretName: es-public-tls
  rules:
  - host: ${ES_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: es-es-http
            port:
              number: 9200
```
  - check certificate:
    - kubectl -n elasticsearch get certificate,order,challenge
    - kubectl -n elasticsearch describe certificate es-public-tls
    - kubectl -n elasticsearch get certificate es-public-tls -o jsonpath='{.status.notBefore}{"\n"}{.status.notAfter}{"\n"}{.status.renewalTime}{"\n"}'

  <img width="898" height="752" alt="image" src="https://github.com/user-attachments/assets/8e215f92-d433-42f4-983b-c68ff7d1fe4c" />

  - https://es.89.167.16.69.nip.io/

