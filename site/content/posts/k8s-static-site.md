---
title: "Deploying a static site on Kubernetes"
date: 2021-12-16
---

Alright now that we've gotten the boring _first-blog-here's-how-I-created-my-website-post_ post out of the way, I can actually talk about something cool.

Deploying a static website on Kubernetes (k3s in this case) is rather easy. A few benefits of running a static site are the minimal dependencies and no frills deployment. Which in turn results in extremely simple manifests.

Create a minimal nginx:alpine based docker image.
{{< highlight docker >}}
FROM nginx:alpine
COPY site/public /usr/share/nginx/html
{{< /highlight >}}

Build and push that image to a remote registry.

Namespace spec.
{{< highlight yaml >}}
apiVersion: v1
kind: Namespace
metadata:
  name: cbull
{{< /highlight >}}

Create a basic deployment spec.

{{< highlight yaml >}}
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: cbull-dev
  name: cbull-dev
  labels:
    app: cbull-dev
spec:
  selector:
    matchLabels:
      app: cbull-dev
  template:
    metadata:
      labels:
        app: cbull-dev
    spec:
      containers:
        - name: cbull-dev
          image: csbull55/cbull-dev:main-latest
{{< /highlight >}}

Create a service to expose it as a ClusterIP. Note the selector on both the deployment and service spec. Our nginx base image exposes port 80 by default and the targetPort defaults to whatever the port is set to, but might as well make it explicit.
{{< highlight yaml >}}
apiVersion: v1
kind: Service
metadata:
  name: cbull-dev
  namespace: cbull-dev
  labels:
    app: cbull-dev
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: cbull-dev
  sessionAffinity: None
  type: ClusterIP
{{< /highlight >}}

Create an ingress with our service as our backend target.

{{< highlight yaml >}}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cbull-dev
  namespace: cbull-dev
  labels:
    app: cbull-dev
  annotations:
    acme.cert-manager.io/http01-ingress-class: external
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  ingressClassName: external
  rules:
    - host: cbull.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              serviceName: cbull-dev
              servicePort: 80
  tls:
    - hosts:
      - cbull.dev
      secretName: cbull-dev-acme-certificate
{{< /highlight >}}

Lastly make a dns entry with my domain provider that points `cbull.dev` to the external ingress controller I've selected in the ingress spec.

Sit back and relax as cert-manager handles the entire interaction of requesting a cert with letsencrypt. It even sets up a temp response pod to answer the domain verification challenge. On re-deployments it checks if the cert secret exists and is valid, and requests another if that's not the case. Although manually deleting the secret resource can force another request.

It's worth noting that letsencrypt does have rate limits on their production instance - https://letsencrypt.org/docs/rate-limits/. So I'd recommend using their staging environment when testing. Once ready, simply adjust the annotation and delete the secret to request another.
