


## Deploying a Container to Kubernetes

**In this section**, we'll run our first container on Kubernetes. We'll work with a docker image that already has content in it. One of the things that makes containers easy and helpful is you can obtain images with software that's been pre-packaged and ready to run... that simplifies the usual effort to install and configure software.

`NOTE: Your USERID is your namespace`

## Task 1: Deploy an Nginx server

In this section we'll deploy an Nginx web server...

`NOTE: Change the namespace to your USERID in the command`
```
$ kubectl run my-deploy --image=nginx:1.7.9 --port=80 --namespace=userxx

deployment "my-deploy" created
```
You can see what you just deployed by issuing the **get deploy** command.

`NOTE: Change the namespace to your USERID in the command`
```
$ kubectl get deploy --namespace=userxx

NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-deploy   1         1         1            1           1m
$
```
You can see the pod you created issuing the **get pods** command

`NOTE: Change the namespace to your USERID in the command`
```
$ kubectl get pods --namespace=userxx
NAME                         READY     STATUS        RESTARTS   AGE
my-deploy-dbc4b8b5d-krhwh    1/1       Running       0          1m

```

Now we have a container running...we'll need to provide access to the server by creating a [service](https://kubernetes.io/docs/concepts/services-networking/service/) to make a port available.

`NOTE: Change the namespace to your USERID in the command`
```
$ kubectl expose deployment my-deploy  --type=NodePort --namespace=userxx
service "my-deploy" exposed
$
$ kubectl get service --namespace=user99-ns
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          1d
my-deploy    LoadBalancer   10.111.227.3    <pending>     80:30839/TCP     1m


```

The port is open, so the nginx server should be available... now, we'll use the ip of the cluster, and the port we've exposed to access the server.

`NOTE: Use http to access your app (NOT https)`

Then we can assemble the url as http:// + [icp master ip] + ":" + [port assigned by the service].
If the ip address for the master of our cluster is 192.168.99.100 then we should use http://192.168.99.100:30839 to access our new Nginx server.

You can either use the browser or the curl command below. Both should work!

`Note: The master ip address is the ip address you used to log into the console, not the ip address listed in the previous command.`

```
$ curl http://192.168.99.100:30839
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

...
</html>
```

## Task 2: Scale the Nginx deployment and test its resilience

The server is deployed, but what if we get a lot more traffic than we expected? Fortunately we can manage more instances of our Nginx pods to handle the additional traffic. We'll use the kubectl client to scale our deployment so it has 5 copies running instead of just one.

`NOTE: Change the namespace to your USERID in the command`

```
$ kubectl scale deploy my-deploy --replicas=5 --namespace=userxx
deployment "my-deploy" scaled
```

`NOTE: Change the namespace to your USERID in the command`

```
$ kubectl get deploy --namespace=userxx
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-deploy   5         5         5            5           58s

```

You should now see 5 instances of your pod running

`NOTE: Change the namespace to your USERID in the command`
```
$ kubectl get pods --namespace=userxx
NAME                         READY     STATUS    RESTARTS   AGE
my-deploy-6fd857889c-8pm6x   1/1       Running   0          50s
my-deploy-6fd857889c-ht7pv   1/1       Running   0          1m
my-deploy-6fd857889c-kkx5k   1/1       Running   0          50s
my-deploy-6fd857889c-nmccq   1/1       Running   0          50s
my-deploy-6fd857889c-xqvqz   1/1       Running   0          50s

```

Our workloads on Kubernetes are quite resilient if we let the system handle things for us. Let's see how resilient our workload is by deleting a few pods!

`NOTE: Change the namespace to your USERID in the command`

`NOTE: Change the pod NAME to match any one of your pods`

```
$ kubectl delete pod my-deploy-6fd857889c-xqvqz --namespace=userxx
pod "my-deploy-6fd857889c-xqvqz" deleted

```
The pod is deleted as you requested... but a new pod has been created in its place! Best of all... the service continues to keep track of all the appropriate pods and route network traffic to them as soon as they're available.

```
$ kubectl get pods --namespace=userxx
NAME                         READY     STATUS        RESTARTS   AGE
my-deploy-6fd857889c-8nvf5   1/1       Running       0          12s
my-deploy-6fd857889c-8pm6x   1/1       Running       0          3m
my-deploy-6fd857889c-dzwlf   1/1       Running       0          12s
my-deploy-6fd857889c-ht7pv   1/1       Running       0          4m
my-deploy-6fd857889c-kkx5k   1/1       Running       0          3m
my-deploy-6fd857889c-xqvqz   0/1       Terminating   0          3m
```


## Task 3: Update the pod with a new version of the software

It's great that the system scales now... but what happens when we have new versions of our software to deploy? To do that we can simply update the existing deployment and tell it that there's a new version of the image currently in use that should be rolled out.  

```
$ kubectl set image deployment/my-deploy my-deploy=nginx:1.9 --namespace=userxx
deployment "my-deploy" image updated

```

When we tell kubernetes that our deployment needs to update a new container image it terminates the old versions while rolling out the new version, and ensures that the service will re-route traffic to the proper pods.

```

$ kubectl get pods --namespace=userxx
NAME                         READY     STATUS        RESTARTS   AGE
my-deploy-6fd857889c-8nvf5   0/1       Terminating   0          17m
my-deploy-6fd857889c-8pm6x   0/1       Terminating   0          20m
my-deploy-6fd857889c-dzwlf   0/1       Terminating   0          17m
my-deploy-6fd857889c-ht7pv   0/1       Terminating   0          21m
my-deploy-6fd857889c-kkx5k   0/1       Terminating   0          20m
my-deploy-857bc7b484-4j684   1/1       Running       0          11s
my-deploy-857bc7b484-mx75q   1/1       Running       0          12s
my-deploy-857bc7b484-phfct   1/1       Running       0          10s
my-deploy-857bc7b484-ttspq   1/1       Running       0          13s
my-deploy-857bc7b484-w27c2   1/1       Running       0          13s

```

To validate that the proper pod is running, do a 'describe' on the deployment or one of the pods and see what image is currently in use...

Let's save the results of our deployment, just in case we want to recreate it

```
kubectl get deployment my-deploy -o=yaml --export --namespace=userxx > ./apache-deploy.yaml
```
You can now view the YAML file on your local machine to ensure that it was updated to the new version

Now that we've gotten a bit of a feel for working with Kubernetes we can clean up and move on to the next thing!

```
$ kubectl delete deployment my-deploy --namespace=userxx
deployment "my-deploy" deleted

```



---
