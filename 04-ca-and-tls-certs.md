# Provisioning A CA And Generating TLS Certificates

## Certificate Authority
The Cert Authority is used to generate additional TLS certificates

Generate the CA configuration file, certificate, and private key:
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
Results:
```
2022/11/23 14:40:11 [INFO] generating a new CA key and certificate from CSR
2022/11/23 14:40:11 [INFO] generate received request
2022/11/23 14:40:11 [INFO] received CSR
2022/11/23 14:40:11 [INFO] generating key: rsa-2048
2022/11/23 14:40:11 [INFO] encoded CSR
2022/11/23 14:40:11 [INFO] signed certificate with serial number 132835444346935663889725582433475573618081639426
```

## Client And Server Certificate
Generating client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user.

The Admin Client Certificate

Generate the admin client certificate and private key:
```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```
results:
```
2022/11/23 14:49:08 [INFO] generate received request
2022/11/23 14:49:08 [INFO] received CSR
2022/11/23 14:49:08 [INFO] generating key: rsa-2048
2022/11/23 14:49:08 [INFO] encoded CSR
2022/11/23 14:49:08 [INFO] signed certificate with serial number 188198165131726128034081163567922386739945318037
2022/11/23 14:49:08 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## The Kubelet Client Certificate
Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by Kubelets. In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the system:nodes group, with a username of system:node:<nodeName>. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:
```
for i in 0 1 2; do
  instance="worker-${i}"
  instance_hostname="ip-10-0-1-2${i}"
  cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance_hostname}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  internal_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PrivateIpAddress')

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance_hostname},${external_ip},${internal_ip} \
    -profile=kubernetes \
    worker-${i}-csr.json | cfssljson -bare worker-${i}
done
```
results:
```
2022/11/23 15:10:38 [INFO] generate received request
2022/11/23 15:10:38 [INFO] received CSR
2022/11/23 15:10:38 [INFO] generating key: rsa-2048
2022/11/23 15:10:39 [INFO] encoded CSR
2022/11/23 15:10:39 [INFO] signed certificate with serial number 529085799452069489067514379761338449680989769228
2022/11/23 15:10:43 [INFO] generate received request
2022/11/23 15:10:43 [INFO] received CSR
2022/11/23 15:10:43 [INFO] generating key: rsa-2048
2022/11/23 15:10:43 [INFO] encoded CSR
2022/11/23 15:10:43 [INFO] signed certificate with serial number 639790309557007681649288037514323383307983593680
2022/11/23 15:10:47 [INFO] generate received request
2022/11/23 15:10:47 [INFO] received CSR
2022/11/23 15:10:47 [INFO] generating key: rsa-2048
2022/11/23 15:10:47 [INFO] encoded CSR
2022/11/23 15:10:47 [INFO] signed certificate with serial number 396494604865302137825266587333019642359311932281
```

## The Controller Manager Client Certificate
Generate the kube-controller-manager client certificate and private key:

```
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

results
```
2022/11/23 15:13:07 [INFO] generate received request
2022/11/23 15:13:07 [INFO] received CSR
2022/11/23 15:13:07 [INFO] generating key: rsa-2048
2022/11/23 15:13:08 [INFO] encoded CSR
2022/11/23 15:13:08 [INFO] signed certificate with serial number 616692190128630942609093782536834873070665561776
2022/11/23 15:13:08 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## The Kube Proxy Client Certificate
Generate the kube-proxy client certificate and private key:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

results:
```
2022/11/23 15:15:00 [INFO] generate received request
2022/11/23 15:15:00 [INFO] received CSR
2022/11/23 15:15:00 [INFO] generating key: rsa-2048
2022/11/23 15:15:00 [INFO] encoded CSR
2022/11/23 15:15:00 [INFO] signed certificate with serial number 669841291948076703910916102062555631656830596077
2022/11/23 15:15:00 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## The Scheduler Client Certificate
Generate the kube-scheduler client certificate and private key:

```
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
results:
```
2022/11/23 15:15:52 [INFO] generate received request
2022/11/23 15:15:52 [INFO] received CSR
2022/11/23 15:15:52 [INFO] generating key: rsa-2048
2022/11/23 15:15:52 [INFO] encoded CSR
2022/11/23 15:15:52 [INFO] signed certificate with serial number 235113929841129661194332005284733622901340643288
2022/11/23 15:15:52 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```
## The Kubernetes API Server Certificate

The kubernetes-the-hard-way static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:
```
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.0.1.10,10.0.1.11,10.0.1.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
results:
```
2022/11/23 15:22:07 [INFO] generate received request
2022/11/23 15:22:07 [INFO] received CSR
2022/11/23 15:22:07 [INFO] generating key: rsa-2048
2022/11/23 15:22:07 [INFO] encoded CSR
2022/11/23 15:22:07 [INFO] signed certificate with serial number 48750249904619473641262075385073580529275793625
```

## The Service Account Key Pair
The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the managing service accounts documentation.

Generate the service-account certificate and private key:
```
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```
results:
```
2022/11/23 15:24:17 [INFO] generate received request
2022/11/23 15:24:17 [INFO] received CSR
2022/11/23 15:24:17 [INFO] generating key: rsa-2048
2022/11/23 15:24:17 [INFO] encoded CSR
2022/11/23 15:24:17 [INFO] signed certificate with serial number 539762335657417056323789531752332783226789266239
2022/11/23 15:24:17 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

## Distribute the Client and Server Certificates
Copy the appropriate certificates and private keys to each worker instance:
```
for instance in worker-0 worker-1 worker-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/
done
```
Copy the appropriate certificates and private keys to each controller instance:

```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')

  scp -i kubernetes.id_rsa \
    ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ubuntu@${external_ip}:~/
done
```