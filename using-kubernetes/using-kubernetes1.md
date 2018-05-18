# Objectives

- Write a Pod manifest and deploy it
- Create a deployment 
- Scale, upgrade and rollback a deployment

## Create an pod

First, create a namespace. This enables you to work in an isolated environment.

`kubectl create namespace $NAMESPACE`

Create your first kubernetes manifest file.


**pod.yaml**
```
cat <<EOT >> pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.12.0
    ports:
    - containerPort: 80
EOT
```

Deploy the pod onto the cluster to the namespace you just created:

`kubectl create -f pod.yaml --namespace $NAMESPACE`


See if the pod is running

`kubectl get pods --namespace $NAMESPACE`

`STATUS: Running` and `READY 1/1` indicates that the pod is ready.


Just like with docker itself, with `kubectl` you can directly attach yourself to a container

`kubectl exec -it $POD $COMMAND -c $CONTAINER --namespace $NAMESPACE`

Try out if you can attach yourself to the pod you created in the last step.

Delete the pod with `kubectl delete pod`

Watch the change of the status of the pod with `watch kubectl get pods --namespace $NAMESPACE`

## Create a deployment 

**deployment.yaml**
```
cat <<EOT >> deployment.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: $DEPLOYMENT_NAME
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12.0
        ports:
        - containerPort: 80
EOT
```

Create the deployment:

`kubectl create -f deployment.yaml --namespace $NAMESPACE` 

Check the status of the deployment and if the pod is running

```
kubectl get deployments --namespace $NAMESPACE
kubectl get pods --namespace $NAMESPACE
```
Open a second console and watch what happens:
`watch kubectl get pods --namespace $NAMESPACE`

then:
 `kubectl delete pod nginx --namespace $NAMESPACE`


## Scale, upgrade and rollback an deployment
Look at your current deployment in detail with 
`kubectl describe deployment $DEPLOYMENT_NAME --namespace $NAMESPACE`

Especially pay attention to:
- `StrategyType` and `RollingUpdateStrategy`
- `Pod Template`
- `Events`

#### scale
`kubectl scale deployment $DEPLOYMENT_NAME --replicas $NO_OF_PODS --namespace $NAMESPACE`
See if all pods are ready with 

`kubectl get pods --namespace $NAMESPACE`

#### update
Use kubectls `set image` function to update your deployment with a newer version nginx:

`kubectl set image deployment/nginx-deployment nginx=nginx:1.13.0`

Don't forget to watch the pods of your deployment with `watch kubectl get pods --namespace $NAMESPACE`

Question: Why is a pod with a new image created before an old one is deleted?

#### rollback
If you deployed a version of an app which has a critical bug or you just want to use the last version again, you can do this with kubernetes by rolling to a previous revision of the Deployment.

First, see which revisions of the deployment are available use:

`kubectl rollout history deployment/$DEPLOYMENT_NAME --namespace $NAMESPACE`

You should see a list with of all revisions with its corresponding cause.

Rollback to the desired revision:

`kubectl rollout history deployment/$DEPLOYMENT_NAME --revision=$DESIRED_REVISION --namespace $NAMESPACE`


