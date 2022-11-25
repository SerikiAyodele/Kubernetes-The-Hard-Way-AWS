# Generating Kubernetes Configuration Files for Authentication
Generating kubernetes configuration files also known as kubeconfigs to enable kubernetes clients to locate and authenticate to the kubernetes API server

## Client Authentication Configs
Generate kubeconfig files for the controller manager, kubelet, kube-proxy, and scheduler clients and the admin user.

### Kubernetes Public DNS Address
Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the kubernetes-the-hard-way DNS address:
```
KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${LOAD_BALANCER_ARN} \
  --output text --query 'LoadBalancers[0].DNSName')
```