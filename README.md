# eirinix-sample

This is a sample extension for Eirinix

## Requirements

Go >= 1.12

## How to use

Simply build the project and run it.

    $> git clone https://github.com/SUSE/eirinix-sample
    $> cd eirinix-sample
    $> go build

### External from a kube cluster

    $> ./eirinix-sample --kubeconfig "path/to/my/kubeconfig" --namespace "eirini" --operator-webhook-host "my-ip-reacheable-from-the-cluster" --operator-webhook-port 8889

### In-cluster

You can run the extension from an application pod - just don't specify the kubeconfig as an option, it will connect with in-cluster credentials.

    $> ./eirinix-sample --namespace "eirini" --operator-webhook-host "my-ip-reacheable-from-the-cluster" --operator-webhook-port 8889

The Host/Port where the extension is binding, needs to be accessible by the kubeapi server. One way to do this, for e.g. is creating a new service and re-use the ```ClusterIP``` as ```--operator-webhook-host``` argument.

## What does it do?

It does a really simple thing just to show off the usage of the library: the extension will add an environment variable to Eirini apps pushed in CF.

## Source code

The relevant part of this extension is split into two files: ```helloworld/hello.go``` and ```main.go```.

The logic of the Eirini extension is defined in ```helloworld/hello.go```, here we control what the extension will change to Eirini Application PODs.

In ```main.go``` you can find the startup functions, which are parsing the parameters from the CLI and running the daemon handled by [Eirinix](https://godoc.org/github.com/SUSE/eirinix#Manager).

## Step by step example with Minikube (local)

In this example we will run the extension in our host, and we will use minikube to setup a local kubernetes cluster. It is a quick way to verify that our extension works correctly. 

Requirements:

* Minikube
* Go >= 1.12

### 1) Clone the repository and build the extension

    $> git clone https://github.com/SUSE/eirinix-sample
    $> cd eirinix-sample
    $> go build

### 2) Start minikube

    $> minikube start

### 3) Run the extension

    $> ./eirinix-sample -c $HOME/.kube/config -n default -w 10.0.2.2 -p 3000

To verify the creation of the mutating webhook generated by the Eirinix library:

    $> kubectl get mutatingwebhookconfiguration

    NAME                             CREATED AT
    eirini-x-helloworld-mutating-hook-default   2019-06-10T06:58:12Z


**Note** If you change the location where your extension is being run (local vs in-cluster) on the same cluster, you need to delete the secret and the mutatingwebhookconfiguration generated by the extension (see "Extension resources" below)

### 4) Create a fake Eirini app POD

    $> kubectl create -f contrib/fakes/eirini_app.yaml

### 5) Verify that the environment variable was set by the extension

    $> kubectl describe pod eirini-fakeapp

    ...

    Containers:
        busybox:
            Container ID:  docker://d485cfdfaf023d29462dc8f42ab929499f7a8f22ffa3c45680a91ced8655806b
            Image:         busybox:1.28.4
            Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
            Port:          <none>
            Host Port:     <none>
            Command:
            sleep
            3600
            State:          Running
            Started:      Mon, 10 Jun 2019 08:59:35 +0200
            Ready:          True
            Restart Count:  0
            Environment:
            STICKY_MESSAGE:  Eirinix is awesome!
            Mounts:
            /var/run/secrets/kubernetes.io/serviceaccount from default-token-djbhs (ro)
        Conditions:

    ...

## Minikube in-cluster example

The following steps assumes you have the repository cloned and that is your working directory, also the minikube cluster has to be already up and running.

### 1) Build the docker image

    $> eval $(minikube docker-env)
    $> docker build --rm -t eirinix-sample-extension .

Or build with Cloud Native Buildpacks:

    $> pack build eirinix-sample-extension --builder cloudfoundry/cnb:bionic

### 2) Create the service

    $> kubectl apply -f contrib/eirinix-sample-extension-service.yaml

### 3) Set up role permissions

Ideally we need to create a role for our extension and bind to that one. The role should be able to operate over namespace labels and pods (at least), see [here](https://github.com/cloudfoundry-incubator/cf-operator/blob/master/deploy/helm/cf-operator/templates/role.yaml) for a full example.

Set up the roles/serviceaccount:

    $> kubectl apply -f contrib/role.yaml
    role.rbac.authorization.k8s.io/eirini-helloworld-extension created 
    $> kubectl apply -f contrib/service_account.yaml
    serviceaccount/eirini-helloworld-extension created
    $> kubectl apply -f contrib/cluster_role.yaml
    clusterrole.rbac.authorization.k8s.io/eirini-helloworld-extension created
    clusterrolebinding.rbac.authorization.k8s.io/eirini-helloworld-extension created
    $> kubectl apply -f contrib/role_binding.yaml
    rolebinding.rbac.authorization.k8s.io/eirini-helloworld-extension created

Or in alternative, for testing you can also set broader permissions, and drop ```serviceAccountName``` from ```contrib/eirinix-sample.yaml```:

    $> kubectl create clusterrolebinding default --clusterrole=cluster-admin --user=system:serviceaccount:default:default

### 4) Run the Extension pod

Now that the roles are setted up, we can run the Extension pod

    $> kubectl apply -f contrib/eirinix-sample-extension.yaml

Verify also that the extension is running:

    $> kubectl logs eirini-helloworld-extension
    2019-06-10T13:12:46.130Z        INFO    eirinix-sample/main.go:27     Starting 0.0.1 with namespace default
    2019-06-10T13:12:46.141Z        INFO    config/getter.go:56     Using in-cluster kube config
    2019-06-10T13:12:46.142Z        INFO    config/checker.go:36    Checking kube config
    2019-06-10T13:12:46.194Z        INFO    ctxlog/context.go:51    Creating webhook server certificate
    2019-06-10T13:12:46.194Z        DEBUG   in_memory_generator/certificate.go:21   Generating certificate webhook-server-ca
    2019-06-10T13:12:53.833Z        DEBUG   in_memory_generator/certificate.go:21   Generating certificate webhook-server-cert

Check also that the mutating webhook is in place:

    $> kubectl get mutatingwebhookconfiguration
    NAME                                        CREATED AT
    eirini-x-helloworld-mutating-hook-default   2019-06-10T13:12:58Z

### 5)  Create a fake Eirini app POD

    $> kubectl create -f contrib/fakes/eirini_app.yaml

### 6) Verify that the environment variable was set by the extension

    $> kubectl describe pod eirini-fake-app

    ...

    Containers:
    eirini-fake-app:
        Container ID:  docker://292290d4f6db4f623038ac38158012480f5da43039ad1278f8f0199e696e39cc
        Image:         busybox:1.28.4
        Image ID:      docker-pullable://busybox@sha256:141c253bc4c3fd0a201d32dc1f493bcf3fff003b6df416dea4f41046e0f37d47
        Port:          <none>
        Host Port:     <none>
        Command:
        sleep
        3600
        State:          Running
        Started:      Mon, 10 Jun 2019 15:15:02 +0200
        Ready:          True
        Restart Count:  0
        Environment:
        STICKY_MESSAGE:  Eirinix is awesome!
        Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-mq9t9 (ro)
    ...

### 7) Celebrate!

## Extension resources

Eirinix registers two resources to kubernetes that allows the kubernetes API to contact our extension.

A new secret containing the generated certificate for the webserver is created if not existing already, and a mutatingwebhookconfiguration as well.

If your plan to run extension(s) in differents pods in a kubernetes cluster, it is needed to define the ```OperatorFingerprint``` ( see [here](https://godoc.org/github.com/SUSE/eirinix#ManagerOptions) ) option to distinguish the extensions sets that are consuming certificates.

You can check for e.g. the secrets generated:

    $>  kubectl get secrets
    NAME                                TYPE                                  DATA   AGE
    eirini-x-helloworld-setupcertificate           Opaque                                4      129m

The default fingerprint for an extension is "_eirini-x_" , and after deploying the extension in the cluster a new secret is automatically generated. In this Eirini extension sample we set it to "eirini-x-helloworld".

The same applies for mutatingwebhookconfiguration

    $> kubectl get mutatingwebhookconfiguration
     NAME                                     CREATED AT
     eirini-x-helloworld-mutating-hook-default   2019-06-10T08:55:30Z

**Note**: If in an existing deployment you are **migrating** the service into a different clusterIP ( or simply, that the binding ip address of the extension would change ) you need to manually delete the resources of the extension before dissming the old POD and let the extension recreate the certificates from scratch during startup. ( you need to run ``` kubectl delete mutatingwebhookconfiguration eirini-x-helloworld-mutating-hook-default``` and ```kubectl delete secret eirini-x-helloworld-setupcertificate``` in our example, if you try to switch between in-cluster and local on the same k8s cluster)
