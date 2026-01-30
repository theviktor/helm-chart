# Deploying jambonz on AWS EKS

## Prerequisites

- AWS account with permissions to create EKS clusters and IAM roles
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [terraform](https://www.terraform.io/downloads)

## Create the Cluster

Use the terraform scripts to create an EKS cluster with the required nodepools:

https://github.com/jambonz-selfhosting/terraform/tree/main/aws/provision-eks-cluster

The terraform will create:
- EKS cluster with default nodepool
- SIP nodepool with appropriate labels, taints, and security groups
- RTP nodepool with appropriate labels, taints, and security groups
- Required IAM roles and policies

After terraform completes, configure kubectl:
```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
```

Verify connectivity:
```bash
kubectl get nodes -o wide
```

## Setting up DNS

Get the load balancer DNS name created by traefik:
```bash
kubectl -n jambonz get svc | grep traefik
```

Or find it in the AWS Console under EC2 > Load Balancers.

Create DNS records in your DNS provider:
- **ANAME/ALIAS record** for your root domain (e.g., `jambonz.example.com`) pointing to the load balancer DNS name
- **CNAME records** for `api`, `grafana`, and `homer` subdomains pointing to the load balancer DNS name

## Post-Installation

After the helm chart is installed, verify that the SIP and RTP security groups are attached to the EC2 instances in the sip and rtp nodepools. This allows external SIP and RTP traffic to reach the cluster.

Return to the [main instructions](../README.md#provision-the-cluster) to continue setup.
