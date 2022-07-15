# This repository is a partial fork of Philip Haberkern's [project](https://github.com/thedatabaseme/cookbooks/tree/master/kubernetes/uptime-kuma)

Oringinal Document: https://thedatabaseme.de/2022/02/12/how-to-deploy-uptime-kuma-on-kubernetes/

<details>
  <summary>Click to expand!</summary>

Quick instruction on how to deploy Uptime-Kuma on your Kubernetes cluster.

Uptime-Kuma is an easy to deploy and easy to use monitoring tool, that has several “Monitoring Types” like HTTP(s), port-checking, DNS and some more. Also it includes some real nice alerting mechanisms as Mail, Teams and Gotify. If you want to find out more about Uptime-Kuma, you can find some here.

I’ve created a set of manifests, that will deploy Uptime-Kuma with all needed components on your Kubernetes cluster. You can find the most updated code in my Github repository. Just apply it using kustomization like `kubectl apply -k overlays/dev`.

If you want to go step-by-step, read along below. You can apply the manifests in the given order after you’ve created them.

First we create a separate Namespace for Uptime-Kuma named kuma. Feel free to change the name to your needs.

namespace.yaml
```
kind: Namespace
apiVersion: v1
metadata:
  name: kuma
```

Then we create a Service for Uptime-Kuma which will listen on port 3001. The selector will be app: uptime-kuma.

service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: uptime-kuma-service
  namespace: kuma
spec:
  selector:
    app: uptime-kuma
  ports:
  - name: uptime-kuma
    port: 3001
```

Now it’s time for the centerpiece our Statefulset which will describe the actual Deployment and a persistent volume.

statefulset.yaml
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: uptime-kuma
  namespace: kuma
spec:
  replicas: 1
  serviceName: uptime-kuma-service
  selector:
    matchLabels:
      app: uptime-kuma
  template:
    metadata:
      labels:
        app: uptime-kuma
    spec:
      containers:
        - name: uptime-kuma
          image: louislam/uptime-kuma:1.11.4
          env:
            - name: UPTIME_KUMA_PORT
              value: "3001"
            - name: PORT
              value: "3001"
          ports:
            - name: uptime-kuma
              containerPort: 3001
              protocol: TCP
          volumeMounts:
            - name: kuma-data
              mountPath: /app/data

  volumeClaimTemplates:
    - metadata:
        name: kuma-data
      spec:
        accessModes: ["ReadWriteOnce"]
        volumeMode: Filesystem
        resources:
          requests:
            storage: 1Gi
```

The Ingress rule that is getting deployed is configured for Nginx as an Ingress Controller. You might need to adapt it to your needs. Also an annotation for Cert-Manager is included. If you don’t have a Cert-Manager setup, you don’t need this annotation. You can find an instruction for both, setting up Nginx and Cert-Manager here and here (both only in german).

Also you have to configure your specific DNS name you want to use for Uptime-Kuma in the host(s) sections.

ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuma
  namespace: kuma
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
  - hosts:
    - uptime-kuma.mydomain.de
    secretName: uptime-kuma.mydomain.de
  rules:
  - host: uptime-kuma.mydomain.de
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: uptime-kuma-service
            port:
              number: 3001
```

After deploying all manifests, you should be able to get connected to Uptime-Kuma (you of course have to configure the DNS name on your Networks DNS Server). On first login, you have to create an admin user. After this is done, you are redirected to the dashboard. Have fun playing around with Uptime-Kuma!

</details>
