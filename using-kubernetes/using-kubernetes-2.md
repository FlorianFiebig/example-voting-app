# Objectives

- Deploy the docker-voting-example-app on kubernetes
- Extend the example app with an InitContainer
- Expose the deployment to the public internet

## Deploy the docker-voting-example-app on kubernetes

Navigate to the folder of the example-voting-app and deploy the apps

`kubectl create -f /k8s-specifications --namespace $NAMESPACE`

delete with `kubectl delete  -f /k8s-specifications --namespace $NAMESPACE` 

and create them again with `kubectl create -f /k8s-specifications --namespace $NAMESPACE`


Watch the pods of your deployment with `watch kubectl get pods --namespace $NAMESPACE`

You should see a restart on the worker node. 

Inspect the logs of the pod to see the reason for failure:

`kubectl logs -p worker-pod`

You should see something like

```
System.TimeoutException: The operation has timed out.
   at Npgsql.NpgsqlConnector.Connect(NpgsqlTimeout timeout)
   at Npgsql.NpgsqlConnector.RawOpen(NpgsqlTimeout timeout)
   at Npgsql.NpgsqlConnector.Open(NpgsqlTimeout timeout)
   at Npgsql.ConnectorPool.Allocate(NpgsqlConnection conn, NpgsqlTimeout timeout)
   at Npgsql.NpgsqlConnection.OpenInternal()
   at Worker.Program.OpenDbConnection(String connectionString) in /code/src/Worker/Program.cs:line 74
   at Worker.Program.Main(String[] args) in /code/src/Worker/Program.cs:line 19
```
The reason for the Exception: The worker node started faster than the database and could not reach it within the timeout. In this case it is not critical that the container of othe worker-pod was restarted but in many cases you want to make sure that all depending services of a service are up and running when you start the service.

This is there the `initContainer` comes in.

edit the `worker-deployment.yaml` in the `k8s-specifications` folder and add:
```
initContainers:    
- name: wait-for-postgres
  image: postgres:9.6.5
  command: ['sh', '-c', 'until pg_isready -h db -p 5432; do echo waiting for database; sleep 2; done;']
```
on the same intendation level as `containers`

deploy the worker: `kubectl create -f worker-deployment.yaml`

Tail the logs of the `initContainer` and inspect the status of the pod with

`kubectl logs -f worker-pod -c wait-for-postgres`

Inspect the status of the worker pod: `kubectl get pod` and notice the `STATUS`


deploy the remaining apps and services with 

`kubectl create -f /k8s-specifications --namespace $NAMESPACE`

As soon as the service is up, the status of the worker-pod changes to ContainerCreating/Running.

## Expose your deployment to the public internet

Deploy the whole example-app with:

List all available services with `kubectl get services`

There are three types of Services:

- `ClusterIP`: Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType.

- `NodePort`: Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting <NodeIP>:<NodePort>. (But only inside the VPC)

- `LoadBalancer`: Exposes the service externally using a cloud provider’s load balancer. NodePort and ClusterIP services, to which the external load balancer will route, are automatically created.

Expose your services with a Load Balancer via:

`kubectl expose deployment vote-lb --type="LoadBalancer" --name=vote-lb --port=5000 --target-port=80`

`kubectl expose deployment result-lb --type="LoadBalancer" --name=result-lb --port=5001 --target-port=80`

Run `kubectl get services` again. The `External IP` of the new services is `pending`, because it may take several minutes for the external IP address to propagate. 

Run `kubectl get services` after waiting 1-2 minutes. 

Publicly reachable address = `http://$EXTERNAL_IP:$PORT`


