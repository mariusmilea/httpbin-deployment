# httpbin-deployment

## Assumptions
We assume we're running a k8s cluster with 3 nodes where we have the following setup:
- external-dns serving 'mydomain.com'
- cert-manager taking care of generating letsencrypt TLS certs
- ingress-nginx for exposing our deployment to the outside world (NOTE: ingress-nginx has been retired and GatewayAPI should be used, but we'll use it here for simplicity)

## Create helm chart and deploy
1. create a helm chart skeleton: 
```bash
helm create httpbin
```

2. edit values.yaml and replace the image repository with kennethreitz/httpbin: latest (only tag 'latest' is available in dockerhub, however it is not a good practice to use tag 'latest')
also update replicaCount to 10

3. uncomment the following lines to enable a few basic security controls:
```bash
securityContext: {}
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```
4. set ingress to true and uncomment annotations and tls 
5. edit httpbin/templates/deployment.yaml and add:
```bash
{{- with .Values.topologySpreadConstraints }}
topologySpreadConstraints:
  {{- toYaml . | nindent 8 }}
{{- end }}
```
6. add topologySpreadConstraints to values.yaml (since we have 10 replicas and 3 k8s nodes, we'll get this distribution across the k8s nodes: 4 3 3):
```bash
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname  # Spread across nodes
  whenUnsatisfiable: ScheduleAnyway
```
7. set limits and resources by uncommenting the lines from below to give a range of min and max for allowed cpu and memory usage:
```bash
resources:
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

The entire helm chart can be templated by running the command from below and checking the output:
```bash
helm template myhttpbin httpbin/
```

and then it can be installed onto a k8s cluster by running:
```bash
helm install myhttpbin httpbin/ -f httpbin/values.yaml --namespace myhttpbin --create-namespace
```

We can now take this helm chart and deploy it in a CI/CD pipeline using helmfile or argoCD.

## Simplest way - development only
If we want to do it in the simplest way possible, as a single command, then we'd just run this:
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myhttpbin
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myhttpbin
  template:
    metadata:
      labels:
        app: myhttpbin
    spec:
      securityContext:
        fsGroup: 1000
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myhttpbin
      containers:
      - name: httpbin
        image: kennethreitz/httpbin:latest
        ports:
        - containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          runAsGroup: 1000
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        readinessProbe:
          httpGet:
            path: /status/200
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /status/200
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
      volumes:
      - name: tmp
        emptyDir: {}
EOF
```

Now, to test our deployment and its pods, we could run the following commands:
```bash
$ kubectl get deploy -n myhttpbin
$ kubectl get pods -n myhttpbin
$ curl httpbin.mydomain.com
```
and to test the development deployment, we'd have to do it from within another container, like so:
```bash
$ kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
$ curl http://POD_IP:8080
```

also useful for monitoring are:
```bash
kubectl top pods
kubectl events
```
