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

## Running apps

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