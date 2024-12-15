# Kubernetes introduction

In this introduction you will learn some basic Kubernetes concepts. A pytorch app is given which needs to be build into a Docker container and deployed on a local Kubernetes cluster. In the end you will have deployed a web server with an endpoint protected via basic auth to predict the contents of an image.

## How to use the local kubernetes cluster

When opening the vscode devcontainer a Kubernetes cluster is created via [kind](https://kind.sigs.k8s.io/). A tool for running local Kubernetes clusters using Docker containers as nodes.

Inside the devcontainer is the CLI tool [`kubectl`](https://kubernetes.io/docs/reference/kubectl/) installed. You will need to use this tool to communicate with the Kubernetes cluster. A useful resource is the [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/), which lists a lot of the basic commands.

👉 Open a terminal inside the devcontainer and try the `kubectl get nodes` command. This should show you two nodes, a control plane and worker node, both in `Ready` state.


## Lab layout

Below you see a filesystem tree of this repository. The relevant files are shown and will be needed or are useful to consult during the lab.

```bash
.
├── README.md
├── kubernetes
│   ├── cm-coco-labels.yaml
│   ├── cm-env-vars.yaml
│   ├── deployment.yaml
│   ├── secret.yaml
│   └── service.yaml
├── lab-containers.code-workspace
├── lab-containers.md
├── lab-kubernetes.md
└── object-detection
    ├── Dockerfile
    ├── app.py
    ├── data
    │   ├── coco_labels.txt
    ├── requirements.txt
    └── weights
        └── ssd300_mAP_77.43_v2.pth
```

- The `kubernetes` folder holds all kubernetes manifests which you will create and deploy on the cluster.
- The `object-detection` folder is a pytorch application which will be deployed on the cluster. The relevant files are:
    1. `app.py` the entrypoint of the application.
    1. `Dockerfile` to build the application container. This is fully given, no modifications are needed.
    1. `weights` folder holds the pretrained models. The model will need to be downloaded before building the app container.
    1. `data/coco_labels.txt` is a file with training labels.

## The pytorch application

In this lab we will be deploying a Single Shot MultiBox Object Detector with PyTorch. The original implementation can be found on [Github](https://github.com/amdegroot/ssd.pytorch). The application code can be found in this workspace in de `object-detection` folder. It is slightly modified from the original code to fit our use case.

It uses the [Flask](https://flask.palletsprojects.com/en/stable/) framework to create and run a web application. By default, Flask runs on port `5000` and it has one endpoint `/predict` which is protected via basic auth.

Before the app can be used, you need to provide a model which will be used for the predictions. Since the file is >100Mb it can not be stored in Github and needs to be downloaded from an S3 bucket.

👉 Navigate to `object-detection/weights` and download the model with `wget https://s3.amazonaws.com/amdegroot-models/ssd300_mAP_77.43_v2.pth`.

## Building the pytorch application

The app is now ready to be build into a Docker container and stored in a container repository. Because the Docker image will be quite large and dockerhub has a limit on how many images can be pulled we are going to use a local container registry. The registry should be running under the container name `kind-registry` and uses port `5001`. The Dockerfile uses a base pytorch image and installs additional python libraries via `apt-get` and `pip`. It then starts the application via `app.py`.

When building the container we therefore can use `localhost:5001/{imageName}:{imageTag}` to push and pull the container. The container repository is also available inside the kind Kubernetes cluster and can be used with `localhost:5001` as the repository.

👉 Build the pytorch application and push it to the local container repository.

:information_source: You can check if the image was added to the container registry by querying the catalog:

```bash
curl -k localhost:5001/v2/_catalog
```

## Making a Kubernetes deployment

To run our application in Kubernetes we need to run it in a pod. Instead of creating a [`Pod`](https://kubernetes.io/docs/concepts/workloads/pods/) directly we will be using a [`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). Deployments manages a set of pods through the use of [`ReplicaSets`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) and provides declarative updates for our pods. Inside a deployment we can specify how our pod looks like, the replication factor, volumes and many other configurations.

👉 Create a `deployment.yaml` file in the `kubernetes` folder to deploy the pytorch application. You can start from the [nginx example in the docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment).

Before moving to the next section you should see a successful deployment like so:
```bash
$ kubectl get deploy,po
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pytorch-deployment   1/1     1            1           1m

NAME                                     READY   STATUS    RESTARTS      AGE
pod/pytorch-deployment-959f6f596-dzxdr   1/1     Running   1             1m
```

:information_source: Usefull `kubectl` commands:
```bash
# View all deployments, pods and services in the default namespace
kubectl get all
# View all deployments, pods and services in a specific namespace
kubectl -n namespace get all
# View all deployments
kubectl get deployments
# Retrieve the yaml formatted manifest of a specific deployment
kubectl get deployment $DEPLOYMENT_NAME -o yaml
# Describe the state of a pod
kubectl describe pod $POD_NAME
```

## Adding a service

Now we have a container running inside a pod but nobody can interact with it. This is where [Services](https://kubernetes.io/docs/concepts/services-networking/service/) come into play.
A service exposes an application running in the cluster behind a single outward-facing endpoint. If your replica count in your deployment manifest is >1, multiple pods are deployment on the cluster and a service will split the traffic across all pods. Pods can come and go, each with their own ip address, a service will keep track on which pods belong to the same service and where traffic needs to be routed to.

There are 4 types of services, you can find a description of all 4 in the [documentation](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types).
For us, we will only use the default [`ClusterIP`](https://kubernetes.io/docs/concepts/services-networking/service/#type-clusterip) service type. This type will make the service only reachable from within the cluster but we can use
`kubectl port-forward` to forward the service to our localhost for testing purposes.

👉 Create a `service.yaml` file in the `kubernetes` folder and create a service for the deployment. You can start from the [example in the docs](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service).

You can test if the service works by sending a curl command to the `/predict` endpoint. In the example below you will need to replace the `cat.jpg` with another image.

```bash
curl -XPOST -u 'ugain:ugain' -F image=@cat.jpg localhost:8080/predict
```

:information_source: Usefull `kubectl` commands:
```bash
# View all services in the default namespace
kubectl get svc
# Port forward the service to localhost
kubectl port-forward svc/service-name port:port
```

## Using environment variables with config maps

A common way of configuring containers is by using environment variables. These can be set directly in the deployment manifest but often it is easier to manage by keeping environment variables in a [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) object. ConfigMaps store non-confidential data in an API object.

The pytorch application supports two environment variables:
- `PORT`: the port number on which the Flask app will run. Default value is 5000.
- `COCO_LABELS_PATH`: the pytorch application uses a [COCO dataset](https://cocodataset.org/#home) for training. This variable sets the path for the file where the app should look for the COCO labels. Default value is `data/coco_labels.txt`.

👉 Create a `cm-env-vars.yaml` file in the `kubernetes` folder and add the `PORT` environment variable. Update the deployment so it uses the ConfigMap to set the environment variables. An example can be found [here](https://kubernetes.io/docs/concepts/configuration/configmap/#using-configmaps-as-environment-variables).

You can check the logs of the pod to check if the environment values are used. The port of the Flask app will be displayed with the following output:
```
* Running on http://10.244.1.2:5000/ (Press CTRL+C to quit)
```

:information_source: Usefull `kubectl` commands:
```bash
# View all ConfigMaps in the default namespace
kubectl get cm
# View the output of a container in a pod
kubectl logs $PODNAME
```

## Mounting a file in a pod

ConfigMaps can also be used to mount as a file inside a pod. If we want to provide our own COCO labels file we can put the contents inside a ConfigMap and mount the ConfigMap as a volume inside the pod.

👉 Mount the `object-detection/data/coco_labels.txt` file via a ConfigMap inside the pod. Use the `COCO_LABELS_PATH` environment variable so the app can load the correct file. An example can be found in the [documentation](https://kubernetes.io/docs/concepts/configuration/configmap/#using-configmaps-as-files-from-a-pod).

The pod logs will print which file path is used.

## Using secrets

When sensitive data needs to be stored, Kubernetes provides the [`Secret`](https://kubernetes.io/docs/concepts/configuration/secret/) object. This is usually used to store passwords, tokens, keys or certificates. They are similar to ConfigMaps but their content is base64 encoded.

:warning: Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.

We will use a secret to configure basic auth access to the application. By default the application sets the admin and password as `ugain:ugain` but provides the ability to overwrite this by setting the environment variables `SECRET_USERNAME` and `SECRET_PASSWORD`. There are multiple [types of secrets](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types). They indicate the purpose of the secret or it uses the generic `Opaque` type. In our use case we need to set a basic auth username and password so we can use the [`kubernetes.io/basic-auth`](https://kubernetes.io/docs/concepts/configuration/secret/#basic-authentication-secret) type.

👉 Create a `secret.yaml` file in the `kubernetes` folder and create a secret which holds a username and password to be used as basic auth for the application. Modify the deployment so it [loads the environment variables from the secret](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables).

:information_source: Usefull `kubectl` commands:
```bash
# View all secrets in the default namespace
kubectl get secrets
# View the secret manifest in yaml format
kubectl get secret -o yaml
# Decoding the secret data
echo -n "$BASE64_ENCODED_DATA" | base64 --decode
```
