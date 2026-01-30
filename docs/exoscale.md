# Deploying jambonz on Exoscale SKS

## Prerequisites

- Exoscale account with permissions to create SKS clusters
- [Exoscale CLI](https://community.exoscale.com/documentation/tools/exoscale-command-line-interface/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [terraform](https://www.terraform.io/downloads)

## Create the Cluster

Use the terraform scripts to create an SKS cluster with the required nodepools:

https://github.com/jambonz-selfhosting/terraform/tree/main/exoscale

The terraform will create:
- SKS cluster with default nodepool
- SIP nodepool with appropriate labels, taints, and security groups
- RTP nodepool with appropriate labels, taints, and security groups

After terraform completes, download the kubeconfig:
```bash
exo compute sks kubeconfig <cluster-name> admin -z <zone> > kubeconfig
export KUBECONFIG=./kubeconfig
```

Verify connectivity:
```bash
kubectl get nodes -o wide
```

## Setting up DNS

Get the load balancer IP:
```bash
kubectl -n jambonz get svc | grep traefik
```

Create DNS records in your DNS provider:
- **A record** for your root domain (e.g., `jambonz.example.com`) pointing to the load balancer IP
- **A records** for `api`, `grafana`, and `homer` subdomains pointing to the same IP

Return to the [main instructions](../README.md#provision-the-cluster) to continue setup.
