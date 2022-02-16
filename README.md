
##  Deploy TLS secured Minio Server on Kubernetes

### Create self-signed SSL certificate
#### Install onessl
```bash
$ curl -fsSL -o onessl https://github.com/kubepack/onessl/releases/download/0.3.0/onessl-linux-amd64 && chmod +x onessl && sudo mv onessl /usr/local/bin/
```
#### Generate CA’s root certificate
```bash
onessl create ca-cert
```
This will create two files `ca.crt` and `ca.key`.
####  Generate certificate
```bash
onessl create server-cert --domains minio-service.demo.svc
```
This will generate two files `server.crt` and `server.key`. Minio server will start TLS secure service if it find `public.crt` and `private.key` files in `/root/.minio/certs/ directory` of the docker container. The public.crt file is concatenation of `server.crt` and `ca.crt` where `private.key` file is only the `server.key` file.

####  Generate `public.crt` and `private.key` file
```bash
$ cat {server.crt,ca.crt} > public.crt
$ cat server.key > private.key
```
Be sure about the order of `server.crt` and `ca.crt`. The order will be server's certificate, any intermediate certificates and finally the CA's root certificate. The intermediate certificates are required if the server certificate is created using a certificate which is not the root certificate but signed by the root certificate. onessl use root certificate by default to generate server certificate if no certificate path is specified by `--cert-dir` flag. Hence, the intermediate certificates are not required here.

We will create a kubernetes secret with this `public.crt` and `private.key` files and mount the secret to `/root/.minio/certs/ directory` of minio container.

Minio server will not trust a self-signed certificate by default. We can mark the self-signed certificate as a trusted certificate by adding public.crt file in `/root/.minio/certs/CAs` directory.

### Create Secret
```bash
$ kubectl create secret generic -n demo minio-server-secret --from-file=./public.crt --from-file=./private.key
$ kubectl label secret minio-server-secret app=minio -n default
```
### Create Persistent Volume Claim
Minio server needs a Persistent Volume to store data. Let’s create a Persistent Volume Claim to request Persistent Volume from the cluster.

`YAML` for PersistentVolumeClaim:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  # This name uniquely identifies the PVC. Will be used in minio deployment.
  name: minio-pvc
  namespace: demo
  labels:
    app: minio
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    # This is the request for storage. Should be available in the cluster.
    requests:
      storage: 2Gi
```

### Create Deployment
Minio deployment creates pod where the Minio server will run.

`YAML` for Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # This name uniquely identifies the Deployment
  name: minio-deployment
  namespace: demo
  labels:
    app: minio
spec:
  selector:
    matchLabels:
      app: minio-server 
  strategy:
    type: Recreate # If pod fail, we want to recreate pod rather than restarting it.
  template:
    metadata:
      labels:
        # Label is used as a selector in the service.
        app: minio-server
    spec:
      volumes:
        # Refer to the PVC have created earlier
      - name: storage
        persistentVolumeClaim:
        # Name of the PVC created earlier
          claimName: minio-pvc
      - name: minio-certs
        secret:
           secretName: minio-server-secret
           items:
           - key: public.crt
             path: public.crt
           - key: private.key
             path: private.key
           - key: public.crt
             path: CAs/public.crt # mark self signed certificate as trusted
      containers:
      - name: minio
        # Pulls the default Minio image from Docker Hub
        image: minio/minio
        args:
        - server
        - --address
        - ":443"
        - /storage
        env:
        # Minio access key and secret key
        - name: MINIO_ACCESS_KEY
          value: "minioacess"
        - name: MINIO_SECRET_KEY
          value: "miniosecret"
        ports:
        - containerPort: 443
          # This ensures containers are allocated on separate hosts. Remove hostPort to allow multiple Minio containers on one host
          hostPort: 443
          # Mount the volumes into the pod
        volumeMounts:
        - name: storage # must match the volume name, above
          mountPath: "/storage"
        - name: minio-certs
          mountPath: "/root/.minio/certs"
```

### Create service
Others pods will access the minio server through a service.

`YAML` for Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: demo
  labels:
    app: minio
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app: minio-server # must match with the label used in the deployment
```



## Prepare Backend for Stash
### Create Backend Secret
```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ echo -n 'minioacess' > AWS_ACCESS_KEY_ID
$ echo -n 'miniosecret' > AWS_SECRET_ACCESS_KEY
$ cat ./ca.crt > CA_CERT_DATA
$ kubectl create secret generic -n demo minio-restic-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./AWS_ACCESS_KEY_ID \
    --from-file=./AWS_SECRET_ACCESS_KEY \
    --from-file=./CA_CERT_DATA
```
### Create Repository
`YAML` for Repository:
```yaml
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    s3:
      endpoint: 'https://minio-service.demo.svc' # Use your own Minio server address.
      bucket: stash-testing  # Give a name of the bucket where you want to backup.
      prefix: demo # Path prefix into bucket where repository will be created.(optional).
    storageSecretName: minio-restic-secret
```
