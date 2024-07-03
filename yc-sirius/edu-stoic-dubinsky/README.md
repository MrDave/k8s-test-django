# Deploying in Yandex Cloud

## Configuring

Before deploying, make sure to connect and configure k8s using Yandex Cloud credentials.

You should have recieved instructions on connecting to Yandex Cloud which among other things specifies the following:

- namespace (e.g. `edu-stoic-dubinsky`) 
- domain (e.g. `edu-stoic-dubinsky.sirius-k8s.dvmn.org`)
- database name
- NodePort to use with Yandex Application Load Balancer

For ease of use, set `kubectl` to use your dedicated namespace:
```sh
kubectl config set-context --current --namespace=<your-namespace>
```
If successful, you will see output such as:
```
Context "yc-sirius" modified.
```
You can further verify that by running `kubectl config view`:
```sh
kubectl config view --minify | grep namespace:
    namespace: edu-stoic-dubinsky
```

## Deploying apps

Currently, test ngninx app is configured to run.

Apply `nginx-deployment.yaml` and `nginx-svc.yaml` manifest files:
```sh
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-svc.yaml
```
Note that you will have to edit the node port number in `nginx.yaml` file according to your Cloud credentials:
```yaml
...
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: nginx-port
      nodePort: 30261  # Change this to port available to you
  type: NodePort
```

Verify that deployment, service and pod are launched:

```sh
kubectl get all
```
You'll see a list similar to this:
```
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-78c9775fd8-njmbk   1/1     Running   0          153m

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/nginx-svc   NodePort   10.98.129.119   <none>        80:30261/TCP   31m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           153m
```

Now the app is accessible via url to Yandex Cloud domain.

### Updating images

The docker images of the django app are created and pushed with GitHub commit hash as the image's tag.

To do this yourself without checking the hash value manually use `"$(git log -1 --pretty=%h)"` when tagging images. The command should be executed from within the cloned repository's folder for `git` command to work. 

Example:
```sh
docker build -t mrdave95/django-site:"$(git log -1 --pretty=%h)"
```

For extra convenience, create a shell script in `backend_main_django` folder:

```sh
#!/bin/bash 

VERSION=$(git log -1 --pretty=%h)
REPO="registry.example.com/my-project:"  # Replace this string with your repo and image name, e.g. "mrdave95/django-site:"
TAG="$REPO$VERSION"
LATEST="${REPO}latest"
BUILD_TIMESTAMP=$( date '+%F_%H:%M:%S' )
docker build -t "$TAG" -t "$LATEST" --build-arg VERSION="$VERSION" --build-arg BUILD_TIMESTAMP="$BUILD_TIMESTAMP" . 
docker push "$TAG" 
docker push "$LATEST"
```

This script will build the image of the current commit and tag it as both "latest" and with the commit's hash value as well as provide timestamps as environmental variables for the image.

This isn't suggested for checked out prevoius commits as the script will overwrite the image with "latest" tag with the new one. You can still build and push images from previous commits manually.

See [gist with script template](https://gist.github.com/MrDave/05719143fc098092cb7b3d7b1b11e3ef) for details.

## Connecting to PostgreSQL database

### Mounting SSL certificate

#### Creating k8s Secret

To connect to database the Pods must have an SSL certificate installed.

There might already be a Secret in your k8s namespace called `pg-root-cert`. If that's the case, skip this step and continue with [mounting the certificate volume](#mounting-the-certificate-volume).

In order to obtain the certificate check Yandex Cloud's [instructions on getting SSL certificate](https://yandex.cloud/en/docs/managed-postgresql/operations/connect#get-ssl-cert) and copy the `save the certificate`link. Then pass it to `base64` to encode the file:

```sh
curl https://storage.yandexcloud.net/cloud-certs/RootCA.pem | base64
```

Copy the output and create a Secret manifest file:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-root-cert
  namespace: <your-namespace>
data:
  root.crt: <base64-encoded output>
```

Apply the secret using `kubectl apply` then continue to next step.

#### Mounting the certificate volume

In app deployment manifest file include `volumes` in template's `spec` and `volumeMounts` in `spec.containers`:

```yaml
...
template:
  spec:
    containers:
      - name: <container-name>
        image: <container-image>
        volumeMounts:
          - mountPath: "/.postgresql"
            name: pg-root-cert
            readOnly: true
    volumes:
      - name: pg-root-cert
        secret:
          secretName: pg-root-cert
          defaultMode: 384
```

This will ensure that every Pod will be created with `/.posgresql/root.crt` file (the SSL certificate) with `0600` permissions needed for PostgreSQL to work.