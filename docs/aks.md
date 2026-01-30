# Deploying jambonz on Azure AKS

## Prerequisites

- Azure account with permissions to create AKS clusters
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [terraform](https://www.terraform.io/downloads)

## Create the Cluster

Use the terraform scripts to create an AKS cluster with the required nodepools:

https://github.com/jambonz-selfhosting/terraform/tree/main/azure

The terraform will create:
- AKS cluster with default nodepool
- SIP nodepool with appropriate labels, taints, and network security rules
- RTP nodepool with appropriate labels, taints, and network security rules

After terraform completes, configure kubectl:
```bash
az aks get-credentials --resource-group <resource-group> --name <cluster-name>
```

Verify connectivity:
```bash
kubectl get nodes -o wide
```

## Setting up DNS

Get the load balancer public IP:
```bash
kubectl -n jambonz get svc

NAME               TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)                    AGE
traefik            LoadBalancer   10.0.x.x     <public-ip>       80:xxxxx/TCP,443:xxxxx/TCP ...
```

Create DNS records in your DNS provider:
- **A record** for your root domain (e.g., `jambonz.example.com`) pointing to the load balancer IP
- **A records** for `api`, `grafana`, and `homer` subdomains pointing to the same IP

Return to the [main instructions](../README.md#provision-the-cluster) to continue setup.
