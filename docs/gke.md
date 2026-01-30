# Deploying jambonz on Google Cloud GKE

## Prerequisites

- Google Cloud account with permissions to create GKE clusters
- [gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [terraform](https://www.terraform.io/downloads)

## Create the Cluster

Use the terraform scripts to create a GKE cluster with the required nodepools:

https://github.com/jambonz-selfhosting/terraform/tree/main/gcp

The terraform will create:
- GKE cluster with default nodepool
- SIP nodepool with appropriate labels, taints, and firewall rules
- RTP nodepool with appropriate labels, taints, and firewall rules
- Required firewall rules for SIP (5060-5061, 8443) and RTP (40000-60000) traffic

After terraform completes, configure kubectl:
```bash
gcloud container clusters get-credentials <cluster-name> --zone <zone> --project <project-name>
```

Verify connectivity:
```bash
kubectl get nodes -o wide
```

## Setting up DNS

Get the load balancer IP address:
1. Go to **Network services > Load balancing** in the GCP console
2. Select the load balancer created by traefik
3. Copy the IP address from the Frontend section

Create DNS records in your DNS provider:
- **A record** for your root domain (e.g., `jambonz.example.com`) pointing to the load balancer IP
- **A records** for `api`, `grafana`, and `homer` subdomains pointing to the same IP

Return to the [main instructions](../README.md#provision-the-cluster) to continue setup.
