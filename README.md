# jambonz

This helm chart is used to deploy a [jambonz](https://jambonz.org) cluster on Kubernetes.

## Cluster Architecture Overview

Running a VoIP platform in Kubernetes requires special configuration due to the nature of SIP and RTP protocols.

### Why Special Nodepools Are Needed

Standard Kubernetes networking doesn't work well for VoIP traffic because:

- **SIP protocol limitations**: SIP messages contain IP addresses in the message headers (e.g Contact header) body (SDP). When traffic passes through standard k8s networking/NAT, the IP addresses in the SIP headers don't match the actual source, breaking call setup.
- **No ingress controller support**: Unlike HTTP, there are no mature ingress controllers for SIP traffic.
- **RTP port requirements**: RTP media streams require predictable port ranges that can be opened in firewalls.

The solution is to run SIP and RTP workloads on dedicated nodepools with **host networking** enabled, allowing pods to use the node's network interface directly.  Creating a cluster with these dedicated nodepools is a firm requirement of running jambonz in Kubernetes.  

> Don't worry: the cluster setup is largely handled by the terraform scripts described in the docs linked below for each supported hosting provider.

### Required Nodepools

| Nodepool | Host Networking | Purpose | Pods |
|----------|-----------------|---------|------|
| Default | No | Core jambonz services | api-server, webapp, feature-server, mysql, redis, grafana, etc. |
| SIP | Yes | SIP signaling | sbc-sip daemonset |
| RTP | Yes | Media processing | sbc-rtp daemonset |

> **Note**: For smaller deployments, SIP and RTP can share a single nodepool. However, separate nodepools are recommended for production.

## Preparing a Kubernetes Cluster

Use the terraform scripts to create your cluster with the required nodepools:

- [AWS EKS](./docs/eks.md)
- [Google Cloud GKE](./docs/gke.md)
- [Azure AKS](./docs/aks.md)
- [Exoscale SKS](./docs/sks.md)

## Provision the Cluster

### Add a namespace for jambonz
```bash
kubectl create namespace jambonz
```

### Add the traefik helm chart and start it
```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace jambonz
```

If you want to view the traefik dashboard you can enable port forwarding for traefik as shown below (optional).
```bash
kubectl -n jambonz port-forward $(kubectl get -n jambonz pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
```
Then go to http://127.0.0.1:9000/dashboard/ in your browser.

### Start jambonz
Install the jambonz helm chart into the jambonz namespace. The example below shows the required parameters - adjust the hostnames to match your domain.

```bash
helm install jambonz --namespace=jambonz \
--set "cloud=aws" \
--set "webapp.hostname=jambonz.example.com" \
--set "api.hostname=api.jambonz.example.com" \
--set "grafana.hostname=grafana.jambonz.example.com" \
--set "homer.hostname=homer.jambonz.example.com" \
.
```

Set `cloud` to your provider: `aws`, `gcp`, `azure`, or `exoscale`.

This will install jambonz into your kubernetes cluster. It will take a few minutes for storage to be provisioned and databases to be initialized before all pods are ready.

## Setting up DNS

After installing traefik and jambonz, you need to configure DNS records pointing to the load balancer. The process varies by cloud provider - see your provider's documentation:

- [AWS EKS DNS Setup](./docs/eks.md#setting-up-dns)
- [Google Cloud GKE DNS Setup](./docs/gke.md#setting-up-dns)
- [Azure AKS DNS Setup](./docs/aks.md#setting-up-dns)
- [Exoscale SKS DNS Setup](./docs/sks.md#setting-up-dns)

## Securing the Web Portals (HTTPS)

At this point the cluster is running but the portals are using HTTP. The first thing you will probably want to do is to modify the configuration so that the portals are running secured over HTTPS.  To do follow these instructions.

First, install cert-manager:
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.crds.yaml
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

You will now see cert-manager pods running in the cert-manager namespace.

Next, edit the values.yaml file:
- set `global.traefik.tls.enabled` to `true`
- set `global.traefik.clusterIssuer` to `letsencrypt-prod`
- set `global.traefik.email` to your email address (shared with letsencrypt)

Upgrade the cluster:
```bash
helm -n jambonz upgrade jambonz .
```

Verify certificates have been issued.  Be patient as this could take 1-3 minutes.
```bash
kubectl -n jambonz get certificate
```

## Log into Portals

### jambonz portal
Go to `https://<your-webapp-hostname>` and log in with user `admin` and password `admin`. You will be prompted to reset the password.

### grafana
Go to `https://<your-grafana-hostname>` and log in with user `admin` and password `admin`. You will be prompted to reset the password.

### Access K8s Pod Logs in Grafana
To access pod logs in Grafana, add Loki as a datasource:
1. Navigate to `Connections -> Datasources`
2. Search for Loki and add it
3. Set the connection URL to: `http://loki-stack.logging.svc.cluster.local:3100`
4. Click `Save and test`

To view logs, go to the `Explore` tab, set the datasource to Loki, and use label filters.

### homer
Homer access is generally not needed since pcaps are available in the jambonz portal under Recent Calls. However, if you need to access it directly you can do so at `https://<your-homer-hostname>` with user `admin` and password `sipcapture`.

## Supporting SIPS over TLS and Secure WebSockets

This section is optional - skip if you only need standard SIP over UDP/TCP.

Since traefik doesn't front SIP traffic, we use a DNS challenge with letsencrypt to generate TLS certificates.

1. Edit values.yaml to set `sbc.sip.ssl.hostname` (e.g., `sip.jambonz.example.com`)

2. Generate the certificate using certbot:
```bash
certbot certonly --manual --preferred-challenges=dns \
  --email your@email.com \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  -d sip.jambonz.example.com \
  -d *.sip.jambonz.example.com
```

3. Add two TXT records when prompted. The key and certificate will be generated:
```
Certificate is saved at: /etc/letsencrypt/live/sip.jambonz.example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/sip.jambonz.example.com/privkey.pem
```

4. Copy the key and certificate into values.yaml under `sbc.sip.ssl.crt` and `sbc.sip.ssl.key`

5. Upgrade the cluster:
```bash
helm -n jambonz upgrade jambonz .
```

The sbc-sip pod will restart with drachtio listening on:
- 5061/tcp (SIP over TLS)
- 8443/tcp (SIP over WSS)

Finally, add DNS A records for the SIP hostname pointing to the public IPs of nodes in the SIP nodepool.
