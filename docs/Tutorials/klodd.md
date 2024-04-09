---
title: Klodd
layout: default
parent: Tutorial
nav_order: 1
---

# Klodd

---

[Klodd](https://klodd.tjcsec.club/) is a service that makes it easy to deploy per-team instances for CTF challenges. 
Though the service is great, the documentation for new CTF developers is less than ideal.

Some CTF challenges, particularly those involving prototype pollution and remote code execution, can potentially allow competitors to interfere with each other, whether out of malice or simply by solving the challenge. In these situations, it can be difficult to deploy challenges in a way that keeps CTFs fun and competitive for everyone. Klodd solves this problem, by giving each team their own deployment that is identical to everyone else's.

## Authentiation

Just a side note, Klodd Auth can be configured through rCTF. View more information about this here: [Klodd Auth](https://klodd.tjcsec.club/install-guide/prerequisites/). I would reccomend using rCTF auth, as it worked seamlessly for me.

**Note:** Kevin explain how you did this here...

## Installation

Install [Docker](https://www.docker.com/get-started/) and [kubectl](https://kubernetes.io/docs/tasks/tools/)

### Docker Registry

Your challenge docker images must be hosted on a docker registry. This can be done through [Docker Hub](https://hub.docker.com/) or by hosting your own registry with `docker run -d -p 5000:5000 --name registry registry:2.7`

**Note:** The exposed port can be whatever you want, just make sure to pull from that port in the future.

Build your image with `docker build -t localhost:5000/image-name:tag .`
Then, push it to the registry with `docker push localhost:5000/image-name:tag`

### Traefik

Klodd depends on [Traefik](https://traefik.io/traefik/) as a ingress controller. Though you can configure Traefik yourself, it is easier to use the default [Helm chart](https://github.com/traefik/traefik-helm-chart).

To install the Traefik with Helm, run the following commands:
1. Ensure you have [Helm v3 > 3.9.0](https://helm.sh/docs/intro/) installed.
2. Add the Traefik Helm repository: `helm repo add traefik https://traefik.github.io/charts`
3. Deploy Traefik: `helm install traefik traefik/traefik`

**Note:** If you wish to restart Traefik, you can run `helm uninstall traefik` to stop the service, and then `helm install traefik traefik/traefik`.

<!-- A domain and corresponding wildcard TLS certificate are required. Klodd will serve individual instances on subdomains of this domain. You are responsible for properly configuring a wildcard DNS record to point these subdomains at your cluster. Ensure that TLS is properly configured in Traefik as well. -->

### Install the Klodd CRDs and ClusterRole

Use the following commands to install the Klodd CRDs and ClusterRole.

```bash
# Install Klodd Challenge CRD
kubectl apply -f https://raw.githubusercontent.com/TJCSec/klodd/master/manifests/klodd-crd.yaml

# Install Klodd ClusterRole
kubectl apply -f https://raw.githubusercontent.com/TJCSec/klodd/master/manifests/klodd-rbac.yaml
```

### Starting Klodd

**Note** In yaml files listed below, you only need to worry about fields with *comments*.

Place the following in a file named `workloads.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: klodd
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: klodd
  namespace: klodd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: klodd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: klodd # 
subjects:
  - kind: ServiceAccount
    name: klodd
    namespace: klodd
---
apiVersion: v1
kind: Secret
metadata:
  name: klodd
  namespace: klodd
type: Opaque
data: # This is customizable. Base64 decode it or view the sample config.yaml below
  config.yaml: Y2hhbGxlbmdlRG9tYWluOiBsb2NhbGhvc3QuZGlyZWN0Cmt1YmVDb25maWc6IGNsdXN0ZXIKcHVibGljVXJsOiBodHRwczovL2tsb2RkLmxvY2FsaG9zdC5kaXJlY3QKcmN0ZlVybDogaHR0cHM6Ly9jdGYudGpjdGYub3JnCnRyYWVmaWs6CiAgaHR0cEVudHJ5cG9pbnQ6IHdlYnNlY3VyZQogIHRjcEVudHJ5cG9pbnQ6IHRjcAogIHRjcFBvcnQ6IDEzMzcKaW5ncmVzczoKICBuYW1lc3BhY2VTZWxlY3RvcjoKICAgIG1hdGNoTGFiZWxzOgogICAgICBrdWJlcm5ldGVzLmlvL21ldGFkYXRhLm5hbWU6IHRyYWVmaWsKICBwb2RTZWxlY3RvcjoKICAgIG1hdGNoTGFiZWxzOgogICAgICBhcHAua3ViZXJuZXRlcy5pby9uYW1lOiB0cmFlZmlrCnNlY3JldEtleTogInJhbmRvbWx5IGdlbmVyYXRlZCBzZWNyZXQga2V5IgpyZWNhcHRjaGE6CiAgc2l0ZUtleTogNkxlSXhBY1RBQUFBQUpjWlZScXlIaDcxVU1JRUdOUV9NWGppWktoSQogIHNlY3JldEtleTogNkxlSXhBY1RBQUFBQUdHLXZGSTFUblJXeE1aTkZ1b2pKNFdpZkpXZQo=
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: klodd
  namespace: klodd
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: klodd
  template:
    metadata:
      labels:
        app.kubernetes.io/name: klodd
    spec:
      serviceAccountName: klodd # 
      volumes:
        - name: config
          secret:
            secretName: klodd
      containers:
        - name: klodd
          image: ghcr.io/tjcsec/klodd:master # 
          volumeMounts:
            - name: config
              mountPath: /app/config/
              readOnly: true
          ports:
            - name: public
              containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: klodd
  namespace: klodd
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: klodd
  ports:
    - name: public
      port: 5000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: klodd
  namespace: klodd
spec:
  rules:
    - host: klodd.localhost.direct
      http:
        paths:
          - backend:
              service:
                name: klodd
                port:
                  number: 5000
            path: /
            pathType: ImplementationSpecific
```

View the example base64 decoded `config.yaml` below:

```yaml
challengeDomain: localhost.direct # use localhost.direct for local testing, or your domain for production
kubeConfig: cluster
publicUrl: https://klodd.localhost.direct
rctfUrl: https://b01lersc.tf # Your rCTF URL here.
traefik: 
  httpEntrypoint: websecure
  tcpEntrypoint: tcp
  tcpPort: 1337
ingress: 
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: traefik
  podSelector:
    matchLabels:
      app.kubernetes.io/name: traefik
secretKey: "randomly generated secret key"
recaptcha: # These are test keys. Replace them with your own keys when moving to production.
  siteKey: 6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI
  secretKey: 6LeIxAcTAAAAAGG-vFI1TnRWxMZNFuojJ4WifJWe
```

To change the `config.yaml` file, base64 encode it and replace the `config.yaml` value in the `workloads.yaml` file.

For local testing, you should only really need to chance the `rctfUrl`. The `challengeDomain` should be `localhost.direct` and the `publicUrl` should be `https://klodd.localhost.direct`.

To deploy Klodd, run the following command:

```bash
kubectl apply -f workloads.yaml
```

Verify that Klodd is running by running the following:
1. `kubectl get namespace`
2. Ensure the `klodd` namespace is present. You should see something like the following:
```bash
NAME              STATUS   AGE
default           Active   31h
klodd             Active   5s
kube-node-lease   Active   31h
kube-public       Active   31h
kube-system       Active   31h
```

At this point, you should be able to go to `https://klodd.localhost.direct/` and see the Klodd frontend.

![Klodd Frontend](../../images/klodd-frontend.png)

### Building a Challenge

If you haven't already, build your image with `docker build -t localhost:5000/image-name:tag .`
Then, push it to the registry with `docker push localhost:5000/image-name:tag`.

* Remember to replace the port with the one you exposed earlier if you decided not to use 5000.

**Remember** In yaml files listed below, you only need to worry about fields with *comments*.

Create a file named `challenge.yaml` with the following contents:

```yaml
apiVersion: "klodd.tjcsec.club/v1"
kind: Challenge
metadata:
  name: test # The name of the resource is also used in the challenge URL. For example, the page for this challenge is accessible at /challenge/test.
spec:
  name: Test Challenge # This is the name displayed on the frontend. It does not have to be related to metadata.name in any way.
  timeout: 10000 # Each instance will be stopped after this many milliseconds.
  pods:
    - name: app # the name of the pod, ensure this and expose.pod match
      ports: 
        - port: 80 # listed port inside the container, ensure this matches the exposed port
      spec:
        containers:
          - name: main
            image: localhost:5000/image-name:tag # The image to run for this pod.
            resources:
              requests:
                memory: 100Mi
                cpu: 75m
              limits:
                memory: 250Mi
                cpu: 100m
        automountServiceAccountToken: false
  expose:
    kind: http
    pod: app # the name of the pod, ensure this and pods.name match
    port: 80 # the port to expose, ensure this matches the listed port in pods.ports.port
  middlewares:
    - contentType:
        autoDetect: false
    - rateLimit:
        average: 5
        burst: 10
```

To deploy the challenge, run the following command:

```bash
kubectl create -f challenge.yaml
```

You should see something like 

```bash
challenge.klodd.tjcsec.club/test created
```

Ensure your challenge is running properly with the following:
1. `kubectl get challenge`
2. Ensure the challenge is present. You should see something like the following:
```bash
NAME  STATUS   AGE
test  Active   5s
```

**Note:** You can delete your challenge with `kubectl delete challenge test`

At this point, you should be able to go to `https://klodd.localhost.direct/challenge/test` and see your challenge. 

Congratulations! You have successfully deployed a challenge with Klodd!

More examples of Klodd deployments can be found here: [Klodd Examples](https://klodd.tjcsec.club/examples/challenges/)

## Common Issues

### Challenge Not Running

If you click the run button and your challenge stays in the "starting" state, there is probably an issue locating your docker image. Ensure that your image is built and pushed to the registry correctly, and that you are using the correct image name and tag in your `challenge.yaml` file.

You can also check `traefik/whoami:latest` as a test image to see if it is a problem with the image. 

**Note** This is on port 80.

### Unknown as the challenge status

If you see "unknown" as the status of your challenge, there is probably a port conflict or matching issue. Ensure you are using the same two ports in your `challenge.yaml` file, and that it matches with your exposed port in the docker image.

### Debugging

You can view the traefik container's logs to see specific errors, or to find the namespace of the problematic pod.