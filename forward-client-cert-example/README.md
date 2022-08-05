# forward-client-cert-example

1. Install kuma

```bash
kumactl install control-plane | kubectl apply -f -
```

2. Enable mTLS

```bash
echo "apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
      - name: ca-1
        type: builtin
        dpCert:
          rotation:
            expiration: 1d
        conf:
          caCert:
            RSAbits: 2048
            expiration: 10y" | kubectl apply -f -
```

3. Create demo namespace with sidecar injection

```bash
echo "apiVersion: v1
kind: Namespace
metadata:
  name: kuma-demo
  labels:
    kuma.io/sidecar-injection: enabled" | kubectl apply -f -
```

4. Deploy an echo server

```bash
echo "apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
  namespace: kuma-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - image: ealen/echo-server:latest
        imagePullPolicy: IfNotPresent
        name: echoserver
        ports:
        - containerPort: 80
        env:
        - name: PORT
          value: \"80\"
---
apiVersion: v1
kind: Service
metadata:
  name: echoserver
  namespace: kuma-demo
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      appProtocol: HTTP
  type: ClusterIP
  selector:
    app: echoserver" | kubectl apply -f -
```

5. Run a curl container to access echo server

```bash
kubectl run mycurlpod --namespace=kuma-demo --image=curlimages/curl -i --tty -- sh
```

6. Execute curl in the container

```bash
curl echoserver
```

7. You should see a response with `x-forwarded-client-cert` header:

```
{"host":{"hostname":"echoserver","ip":"::ffff:127.0.0.1","ips":[]},"http":{"method":"GET","baseUrl":"","originalUrl":"/","protocol":"http"},"request":{"params":{"0":"/"},"query":{},"cookies":{},"body":{},"headers":{"host":"echoserver","user-agent":"curl/7.84.0-DEV","accept":"*/*","x-forwarded-proto":"http","x-request-id":"febba5cf-a6ce-4680-b4c8-5ae1b16ad06f","x-forwarded-client-cert":"By=spiffe://default/echoserver_kuma-demo_svc_80;By=kuma://app/echoserver;By=kuma://k8s.kuma.io/namespace/kuma-demo;By=kuma://k8s.kuma.io/service-name/echoserver;By=kuma://k8s.kuma.io/service-port/80;By=kuma://kuma.io/protocol/http;By=kuma://kuma.io/service/echoserver_kuma-demo_svc_80;By=kuma://pod-template-hash/6944fb9c86;Hash=93075e5780a1df3a88b86473db39eae24c5a70967300fcb522a56abfdd4c75f5;URI=spiffe://default/mycurlpod_kuma-demo_svc;URI=kuma://k8s.kuma.io/namespace/kuma-demo;URI=kuma://kuma.io/instance/mycurlpod;URI=kuma://kuma.io/protocol/tcp;URI=kuma://kuma.io/service/mycurlpod_kuma-demo_svc;URI=kuma://run/mycurlpod"}},"environment":{"PATH":"/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","HOSTNAME":"echoserver-6944fb9c86-8n9tq","NODE_VERSION":"16.16.0","YARN_VERSION":"1.22.19","PORT":"80","KUBERNETES_PORT_443_TCP":"tcp://10.43.0.1:443","ECHOSERVER_SERVICE_PORT":"80","KUBERNETES_SERVICE_PORT_HTTPS":"443","ECHOSERVER_PORT_80_TCP_PORT":"80","ECHOSERVER_PORT_80_TCP_ADDR":"10.43.73.79","KUBERNETES_PORT":"tcp://10.43.0.1:443","KUBERNETES_PORT_443_TCP_ADDR":"10.43.0.1","ECHOSERVER_SERVICE_HOST":"10.43.73.79","ECHOSERVER_PORT_80_TCP_PROTO":"tcp","KUBERNETES_PORT_443_TCP_PORT":"443","KUBERNETES_SERVICE_HOST":"10.43.0.1","KUBERNETES_PORT_443_TCP_PROTO":"tcp","KUBERNETES_SERVICE_PORT":"443","ECHOSERVER_PORT":"tcp://10.43.73.79:80","ECHOSERVER_PORT_80_TCP":"tcp://10.43.73.79:80","HOME":"/root"}}
```
