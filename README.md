# Deploying Automated Nginx apps on Kubernetes

## Installing kubeadm on raspberry pi (ubuntu 20.04 arm64)

### Update kernel flags for containers

```
sudo vi /boot/firmware/cmdline.txt
```

<code>
net.ifnames=0 dwc\_otg.lpm\_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup\_memory=1 cgroup\_enable=memory ipv6.disable=1 
</code>

### Set Static IP

```
sudo vi /etc/netplan/50-cloud-init.yaml
```


```
 \# This file is generated from information provided by the datasource.  Changes\# to it will not persist across an instance reboot.  To disable cloud-init's\# network configuration capabilities, write a file\# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:\#\# network: {config: disabled}network:    ethernets:        eth0:            addresses:            \- 192.168.187.180/24            dhcp4: false            gateway4: 192.168.187.1            nameservers:                addresses:                \- 8.8.8.8                \- 1.1.1.1            optional: true    version: 2 
 ```

### Disable swap, load netfilter kernel modules, and enable IP forwarding

```
sudo swapoff \-asudo sed \-i '/ swap / s/^\\(.\*\\)$/\#\\1/g' /etc/fstab cat \<\<EOF | sudo tee /etc/modules-load.d/containerd.conf overlay br\_netfilter EOF sudo modprobe overlay sudo modprobe br\_netfiltercat \<\<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.confnet.bridge.bridge-nf-call-iptables  \= 1net.ipv4.ip\_forward                 \= 1net.bridge.bridge-nf-call-ip6tables \= 1EOF\# Apply sysctl params without rebootsudo sysctl \--system sudo reboot 
```

### Install containerd, dependencies for kubeadm, and point apt to latest version of kubernetes

```
sudo apt-get update sudo apt-get install containerd \-ysudo apt-get install \-y apt-transport-https ca-certificates curl gpg sudo mkdir \-p /etc/apt/keyrings/curl \-fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg \--dearmor \-o /etc/apt/keyrings/kubernetes-apt-keyring.gpgecho 'deb \[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg\] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.listsudo apt-get update
```

### Install kubeadm

```
sudo apt-get install \-y kubelet kubeadm kubectlsudo apt-mark hold kubelet kubeadm kubectlsudo systemctl enable \--now kubeletsudo kubeadm config images pull
```

### Add kubernetes config to local user profile

```
mkdir \-p $HOME/.kubesudo cp \-i /etc/kubernetes/admin.conf $HOME/.kube/configsudo chown $(id \-u):$(id \-g) $HOME/.kube/config
```

### Join worker nodes

```
sudo kubeadm join 192.168.187.180:6443 \--token 5woo70.y8afzq79unzldcdk \--discovery-token-ca-cert-hash sha256:0ea7bae9cc85decbdd41ef3d69403469e3a52ef7e6b1e24338fd03a951e73451
```

## RBAC

### Create dev user account

```
mkdir devcertcd devcert/openssl genrsa \-out dev.key 2048openssl req \-new \-key dev.key \-out dev.csr \--subj "/CN=dev/O=dev/O=example.org"sudo openssl x509 \-req \-CA /etc/kubernetes/pki/ca.crt  \-CAkey /etc/kubernetes/pki/ca.key  \-CAcreateserial \-days 730 \-in dev.csr \-out dev.crtkubectl config set\-credentials dev \--client-certificate=dev.crt \--client-key=dev.key kubectl config set-context dev-k8s \--cluster=kubernetes \--user=dev kubectl config use-context dev-k8s 
```

### Role

ns-admin-role.yaml

```
apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata:   name: admin rules: \- apiGroups: \["\*"\]   resources: \["\*"\]   verbs: \["\*"\]
```

### Rolebinding

dev-admin-rb.yaml

```
apiVersion: rbac.authorization.k8s.io/v1 kind: RoleBinding metadata:   name: ns-admin subjects: \- kind: User   user: dev   apiGroup: rbac.authorization.k8s.io roleRef:   kind: Role   name: admin   apiGroup: rbac.authorization.k8s.io
```

### Apply Role and Rolebinding

```
kubectl create ns teleport-appnamespace/teleport-app createdkubectl apply \-f ns-admin-role.yaml \-n teleport-app kubectl apply \-f dev-admin-rb.yaml \-n teleport-app
```

## Cert Manager

### Install cert manager

```
kubectl apply \-f https://github.com/cert-manager/cert-manager/releases/download/v1.19.1/cert-manager.yaml
```

### Install ingress controller

```
kubectl create ns ingress-nginx kubectl apply \-f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.3/deploy/static/provider/cloud/deploy.yaml
```

### Create a staging issuer

```
apiVersion: cert-manager.io/v1kind: Issuermetadata:  name: letsencrypt-prodspec:  acme:    \# The ACME server URL    server: https://acme-v02.api.letsencrypt.org/directory    \# Email address used for ACME registration    email: grant.voss@gmail.com    \# The ACME certificate profile    profile: tlsserver    \# Name of a secret used to store the ACME account private key    privateKeySecretRef:      name: letsencrypt-prod    \# Enable the HTTP-01 challenge provider    solvers:      \- http01:          ingress:            ingressClassName: nginx
```

### Create a prod issuer

```
apiVersion: cert-manager.io/v1kind: Issuermetadata:  name: letsencrypt-prodspec:  acme:    \# The ACME server URL    server: https://acme-v02.api.letsencrypt.org/directory    \# Email address used for ACME registration    email: grant.voss@gmail.com    \# The ACME certificate profile    profile: tlsserver    \# Name of a secret used to store the ACME account private key    privateKeySecretRef:      name: letsencrypt-prod    \# Enable the HTTP-01 challenge provider    solvers:      \- http01:          ingress:            ingressClassName: nginx
```

## Metallb

### Install and apply ippool.yaml and l2advertisement.yaml

```
kubectl apply \-f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml ippool.yaml apiVersion: metallb.io/v1beta1 kind: IPAddressPool metadata:   name: default-pool   namespace: metallb-system spec:   addresses:   \- 192.168.187.200-192.168.187.220 l2advertisement.yaml apiVersion: metallb.io/v1beta1 kind: L2Advertisement metadata:   name: default   namespace: metallb-system spec:   ipAddressPools:   \- default-pool 
```

## Nginx app (Air gapped)

### Nginx Cert 

nginx-cert.yaml

```
apiVersion: v1    kind: Secret    metadata:      name: nginx-cert    type: kubernetes.io/tls    data:      tls.crt: | LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUQwRENDQTFhZ0F3SUJBZ0lTQm9PcmE5TlBnRVZiT010Y3I2NXR0L0ZGTUFvR0NDcUdTTTQ5QkFNRE1ESXgKQ3pBSkJnTlZCQVlUQWxWVE1SWXdGQVlEVlFRS0V3MU1aWFFuY3lCRmJtTnllWEIwTVFzd0NRWURWUVFERXdKRgpPREFlRncweU5URXdNVE13TURFd01qRmFGdzB5TmpBeE1URXdNREV3TWpCYU1DSXhJREFlQmdOVkJBTU1GeW91ClozSmhiblIyYjNOekxtUjFZMnRrYm5NdWIzSm5NSFl3RUFZSEtvWkl6ajBDQVFZRks0RUVBQ0lEWWdBRUlPT24KY1IzQ2lxbWRINDVKbFRmeDJiSVpZSGdMYU1uNjV2U3J5dnBiZFlnY0lHejdxYi8yc3lzdFVkeUViay9GMDJKdQoxZEp2dHZJM2lUdHVpYUhmSVlWRUtsRHdqZWR4RnBsRk9IRncxOW4wKzVaeDdESkxta0kwM1VLVzZRTHZvNElDClBUQ0NBamt3RGdZRFZSMFBBUUgvQkFRREFnZUFNQjBHQTFVZEpRUVdNQlFHQ0NzR0FRVUZCd01CQmdnckJnRUYKQlFjREFqQU1CZ05WSFJNQkFmOEVBakFBTUIwR0ExVWREZ1FXQkJRUmJ2bHdQd1RpUFVkU1VrV2FnNVl5Q0dmNwpZekFmQmdOVkhTTUVHREFXZ0JTUERST2k5aTUrMFZCc014ZzRYVm1PSTNLUnlqQXlCZ2dyQmdFRkJRY0JBUVFtCk1DUXdJZ1lJS3dZQkJRVUhNQUtHRm1oMGRIQTZMeTlsT0M1cExteGxibU55TG05eVp5OHdPUVlEVlIwUkJESXcKTUlJWEtpNW5jbUZ1ZEhadmMzTXVaSFZqYTJSdWN5NXZjbWVDRldkeVlXNTBkbTl6Y3k1a2RXTnJaRzV6TG05eQpaekFUQmdOVkhTQUVEREFLTUFnR0JtZUJEQUVDQVRBdEJnTlZIUjhFSmpBa01DS2dJS0FlaGh4b2RIUndPaTh2ClpUZ3VZeTVzWlc1amNpNXZjbWN2TnpBdVkzSnNNSUlCQlFZS0t3WUJCQUhXZVFJRUFnU0I5Z1NCOHdEeEFIY0EKU1p5YmFkNGRmT3o4TnQ3TmgyU211RnV2Q29lQUdkRlZVdnZwNnluZCtNTUFBQUdaMnh6eGxBQUFCQU1BU0RCRwpBaUVBZ2RHczFiM0J6NW1HT3o1RmZsQWR0dStNdmdHT0pBbEpUTWU2cEgvUUdic0NJUURUUS83ajVRdjdJdEdqCjRJZS84M1pyd0lpME93NUdIM1RWbnhjVlFibWZOZ0IyQU1zNDl4V0pmSVNoUkY5YndkMzd5Vzd5bWxuTlJ3cHAKQllXd3l4VERGRmpuQUFBQm1kc2M4YTRBQUFRREFFY3dSUUlnUjdHOGVIdXhONDFFMldqVnRIS1VsajBLc2ZhNwozMFBCYnEyZmtPZXoxOGdDSVFEbXo5Zk5GL0dkQ1hJeTRCK205VmVzUTAvVktMWVBGY0hpUTEvK3ZwaXBqREFLCkJnZ3Foa2pPUFFRREF3Tm9BREJsQWpBc3FPU0hWT3ZjT2lPam9ONXIzQVZmYjVTaGdkcjlDb1B0TjNDMm91SngKTzBXM0l1cHNqODZ1RHd3YVFtRGJHK0lDTVFDdDh1aWlmNFo0ektGeXRsUkRlOHh5TjNHU2FLUVEvazhqaUlISgpFZ1Q4bEZ6WHg3bDBGV29LOVA3NVo2NWZoOHc9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0KLS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVWakNDQWo2Z0F3SUJBZ0lRWTVXVFk4Sk9jSUp4V1JpL3c5ZnRWakFOQmdrcWhraUc5dzBCQVFzRkFEQlAKTVFzd0NRWURWUVFHRXdKVlV6RXBNQ2NHQTFVRUNoTWdTVzUwWlhKdVpYUWdVMlZqZFhKcGRIa2dVbVZ6WldGeQpZMmdnUjNKdmRYQXhGVEFUQmdOVkJBTVRERWxUVWtjZ1VtOXZkQ0JZTVRBZUZ3MHlOREF6TVRNd01EQXdNREJhCkZ3MHlOekF6TVRJeU16VTVOVGxhTURJeEN6QUpCZ05WQkFZVEFsVlRNUll3RkFZRFZRUUtFdzFNWlhRbmN5QkYKYm1OeWVYQjBNUXN3Q1FZRFZRUURFd0pGT0RCMk1CQUdCeXFHU000OUFnRUdCU3VCQkFBaUEySUFCTkZsOGw3YwpTN1FNQXB6U3N2cnU2V3lyT3E0NG9mVFVPVEl6eFVMVXpETU1OTWNoSUpCd1hPaGlMeHh4czBMWGViNUdEY0hiClI2RVRvTWZmZ1Naak85U05IZlk5Z2pNeTl2UXI1L1dXT3JRVFp4aDdhejZOU05ucTN1MnViVDZIVEtPQitEQ0IKOVRBT0JnTlZIUThCQWY4RUJBTUNBWVl3SFFZRFZSMGxCQll3RkFZSUt3WUJCUVVIQXdJR0NDc0dBUVVGQndNQgpNQklHQTFVZEV3RUIvd1FJTUFZQkFmOENBUUF3SFFZRFZSME9CQllFRkk4TkU2TDJMbjdSVUd3ekdEaGRXWTRqCmNwSEtNQjhHQTFVZEl3UVlNQmFBRkhtMFdlWjd0dVhrQVhPQUNJaklHbGoyNlp0dU1ESUdDQ3NHQVFVRkJ3RUIKQkNZd0pEQWlCZ2dyQmdFRkJRY3dBb1lXYUhSMGNEb3ZMM2d4TG1rdWJHVnVZM0l1YjNKbkx6QVRCZ05WSFNBRQpEREFLTUFnR0JtZUJEQUVDQVRBbkJnTlZIUjhFSURBZU1CeWdHcUFZaGhab2RIUndPaTh2ZURFdVl5NXNaVzVqCmNpNXZjbWN2TUEwR0NTcUdTSWIzRFFFQkN3VUFBNElDQVFCbkUwaEdJTktzQ1lXaTBYeDF5Z3hENXFpaEVqWjAKUkkzdFRaejF3dUFUSDNad1lQSXA5N2tXRWF5YW5EMWowY0RoSVl6eTRDa0RvMmpCOEQ1dDBhNnpaV3pscjk4ZApBUUZOaDh1S0prSUhkTFNoeStuVXllWnhjNWJOZU1wMUx1MGdTekU0TWNxZm1OTXZJcGVpd1dTWU85dzgyT2I4Cm90dlhjTzJKVVlpM3N2SElXUm0zKzcwN0RVYkw1MVhNY1kyaVpkbENxNFdhOW5idWszV1RVNGdyNkxZOE16VkEKYURRRzIrNFUzZUo2cVVGMTBiQm5SMXV1VnlEWXM5Umhyd3VjUlZuZnVEajI5Q01MVHNwbE01ZjV3U1Y1aFVwbQpVd3AvdlY3TTR3NGFHdW50NzRrb1g3MW40RWRhZ0NzTC9ZazUrbUFRVTArdHVlMEpPZkFWL1I2dDFrK1hrOXMyCkhNUUZlb3hwcGZ6QVZDMDRGZEc5TStBQzJKV3htRlN0NkJDdWgzQ0VleTNmRTUyUXJqOVlNNzVydHZJanNtLzEKSGwrdS8vV3F4bnUxWlE0anBhK1ZwdVppR09sV3JxU1A5ZW9nZE9oQ0dpc255ZXdXSndSUU9xSzE2d2lHeVplUgp4cy9CZWt3NjV2d1NJYVZrQnJ1UGlUZk1PbzBaaDRnVmE4L3FKZ01iSmJ5cnd3Rzk3ei9QUmdtTEtDRGw4ejNkCnRBMFo3cXE3ZnRhMEdsMjR1eXVCMDVkcUk1SjFMdkF6S3VXZElqVDF0UDhxQ294U0UveHBpeDhoWDJkdDNoKy8KanVqVWdGUEZaMEVWWjB4U3lCTlJGM01ib0dabllYRlV4cE5qVFdQS3BhZ0RISlFtcXJBY0RtV0puTXNGWTNqUwp1MWlndjNPZWZuV2pTUT09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0=      tls.key: | LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JRzJBZ0VBTUJBR0J5cUdTTTQ5QWdFR0JTdUJCQUFpQklHZU1JR2JBZ0VCQkRCL0c0a2JoY3BxWG0ycDcyUVEKajRpckZDMWlXL0NFemhobUxXd3BzUzc1YVNkSTAvbmwvbjJZcThWRkZhZUtGQWVoWkFOaUFBUWc0NmR4SGNLSwpxWjBmamttVk4vSFpzaGxnZUF0b3lmcm05S3ZLK2x0MWlCd2diUHVwdi9hekt5MVIzSVJ1VDhYVFltN1YwbSsyCjhqZUpPMjZKb2Q4aGhVUXFVUENONTNFV21VVTRjWERYMmZUN2xuSHNNa3VhUWpUZFFwYnBBdTg9Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0=
```

### Nginx app manifest

nginx-app.yaml

```
apiVersion: apps/v1kind: Deploymentmetadata:  name: nginx-app  labels:    app: nginx-appspec:  replicas: 1  selector:    matchLabels:      app: nginx-app  template:    metadata:       labels:        app: nginx-app        spec:      volumes:        \- name: tls-volume          secret:            secretName: nginx-cert        \- name: nginx-config-volume          configMap:            name: custom-nginx-config            \- name: nginx-index-volume          configMap:            name: nginx-index      containers:        \- name: nginx          image: nginx:latest           volumeMounts:            \- name: tls-volume              mountPath: /etc/nginx/ssl              readOnly: true            \- name: nginx-config-volume              mountPath: /etc/nginx/nginx.conf              subPath: nginx.conf            \- name: nginx-index-volume              mountPath: /usr/share/nginx/html/index.html              subPath: index.html 
```

### Nginx custom index.html config

```
apiVersion: v1kind: ConfigMapmetadata:  name: nginx-indexdata:  index.html: |    \<\!DOCTYPE html\>    \<html\>    \<head\>      \<title\>Welcome to Teleport App Page\!\</title\>    \</head\>    \<body\>      \<h1\>Hello World\!\</h1\>      \<p\>Certs backed by configmap\</p\>    \</body\>    \</html\> 
```

### Nginx custom config

```
apiVersion: v1    kind: ConfigMap    metadata:      name: custom-nginx-config      namespace: teleport-app     data:      nginx.conf: |-        user nginx;        worker\_processes auto;        events {            worker\_connections 1024;        }        http {            include /etc/nginx/mime.types;            default\_type application/octet-stream;            sendfile on;            keepalive\_timeout 65;                     server {                listen 443 ssl;                server\_name yourdomain.com;                 root /usr/share/nginx/html;                ssl\_certificate /etc/nginx/ssl/tls.crt;                ssl\_certificate\_key /etc/nginx/ssl/tls.key;                }            }
```

### Nginx Service

Nginx-svc.yaml

```
apiVersion: v1kind: Servicemetadata:  name: nginx-servicespec:  selector:    app: nginx-app  ports:    \- protocol: TCP      port: 443      targetPort: 443  type: LoadBalancer 
```

## Nginx App (Cert manager)

### Nginx app manifest

```
apiVersion: apps/v1kind: Deploymentmetadata:  name: nginx-app-cm  labels:    app: nginx-app-cmspec:  replicas: 1  selector:    matchLabels:      app: nginx-app-cm  template:    metadata:       labels:        app: nginx-app-cm        spec:      volumes:        \- name: nginx-index-volume          configMap:            name: nginx-index-cm      containers:        \- name: nginx          image: nginx:latest           volumeMounts:            \- name: nginx-index-volume              mountPath: /usr/share/nginx/html/index.html              subPath: index.html 
```

### Nginx custom index.html

```
apiVersion: v1kind: ConfigMapmetadata:  name: nginx-index-cmdata:  index.html: |    \<\!DOCTYPE html\>    \<html\>    \<head\>      \<title\>Welcome to Teleport App Page\!\</title\>    \</head\>    \<body\>      \<h1\>Hello World\!\</h1\>      \<p\>Certs configured by Cert Manager\</p\>    \</body\>    \</html\> 
```

### Nginx Service

```
apiVersion: v1kind: Servicemetadata:  name: nginx-service-cmspec:  selector:    app: nginx-app-cm  ports:    \- protocol: TCP      port: 80       targetPort: 80 
```

### Nginx Ingress with cert manager

```
apiVersion: networking.k8s.io/v1kind: Ingressmetadata:  name: teleport-ingress-cm  annotations:     cert-manager.io/issuer: "letsencrypt-staging"spec:  ingressClassName: nginx  tls:  \- hosts:    \- teleport-app.grantvoss.duckdns.org    secretName: quickstart-example-tls  rules:  \- host: teleport-app.grantvoss.duckdns.org    http:      paths:      \- path: /        pathType: Prefix        backend:          service:            name: nginx-service-cm            port:              number: 80 
```

## Git repo

### Create new repo on Github

### Clone empty repo and send initial commit to Github

```
git clone git@github.com:grizzle-boss/teleport-app.gitcd teleport-appecho "\# teleport-app" \>\> README.mdgit add README.mdgit commit \-m "first commit"git branch \-M maingit remote add origin git@github.com:grizzle-boss/teleport-app.gitgit push \-u origin main 
```

### Copy all yaml manifests to teleport-app and use new repo

```
git add \*git commit \-m "nginx-app-cm added"git push \-u origin main 
```

## ArgoCD

### Install ArgoCD, update agro svc to use LB and update the admin password

```
kubectl create namespace argocdkubectl config use-context kubernetes-admin@kubernetes kubectl apply \-n argocd \-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml kubectl patch svc argocd-server \-n argocd \-p '{"spec": {"type": "LoadBalancer"}}' kubectl get svc argocd-server \-n argocd \-o=jsonpath='{.status.loadBalancer.ingress\[0\].ip}' brew install argocd argocd admin initial-password \-n argocd argocd login 192.168.187.202 argocd account update-password 
```

### Configure app in ArgoCD UI

![][image1]  
![][image2]

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>
