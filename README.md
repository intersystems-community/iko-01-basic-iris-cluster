## 1. Basic IRIS Cluster – single data node, no compute nodes/gateways/etc

### Content
- [Introduction](#introduction)
- [Versions](#versions)
- [Kubernetes cluster](#kubernetes-cluster)
- [Custom IRIS-based application image](#custom-iris-based-application-image)
- [IKO Installation](#iko-installation)
- [Minimal application installation](#minimal-application-installation)
- [Check an application availability](#check-an-application-availability)
- [Application Installation with custom settings](#application-installation-with-custom-settings)
- [Conclusion](#conclusion)

### Introduction
This is the first part of code samples repositories intended to demonstrate [InterSystems Kubernetes Operator](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO) (IKO) usage.

Code samples repositories are going to highlight the following topics:
1. <ins>[Basic IRIS Cluster – single data node, no compute nodes/gateways/etc](https://github.com/intersystems-community/iko-01-basic-iris-cluster).</ins>
2. IRIS with Compute nodes – single data node and several compute ECP clients.
3. All about filesystems – examples of all the volumes that IRIS uses (wij, journal1, journal2, etc) and recommendations for which to put on fast storage vs cheap storage.
4. Deploying SAM with your IRIS cluster.
5. Deploying IAM with your IRIS cluster.
6. Creating and using hugepages with IRIS.

You can install IRIS or IRIS-based application in Kubernetes using regular yaml-manifests (deployments, statefulsets, services etc.), but it's much simpler to leverage a specifically created Kubernetes Operator which can deploy a desired IRIS-cluster with minimal settings from your side.

You can read about Kubernetes Operator approach at [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), [Kubernetes Operators Explained](https://blog.container-solutions.com/kubernetes-operators-explained) and [How to explain Kubernetes Operators in plain English](https://enterprisersproject.com/article/2019/2/kubernetes-operators-plain-english) pages.


IRIS-based application should be packed in Docker image to be able running in Kubernetes. Also this image should be available from Kubernetes cluster, i.e. published in Docker registry where Kubernetes has an access to.

Let's use DockerHub as such a registry. You'll need an [account at DockerHub](https://hub.docker.com/signup) to be able to push images.

DockerHub account is called `<dockerhub_account>` throughout this readme. Please replace it by an actual value, i.e. by your own DockerHub account name, like here:
```
$ export DOCKERHUB_ACCOUNT=<dockerhub_account> # like 'johndoe'
$ git clone https://github.com/intersystems-community/iko-01-basic-iris-cluster
$ cd iko-01-basic-iris-cluster
$ grep -rl '<dockerhub_account>' . | grep -v README | xargs sed -i "s/<dockerhub_account>/${DOCKERHUB_ACCOUNT}/g"
# Note: on Mac you might run the following command:
$ grep -rl '<dockerhub_account>' . | grep -v README | xargs sed -i '' "s/<dockerhub_account>/${DOCKERHUB_ACCOUNT}/g"
```

We will store an IRIS-based application in Docker registry as a *public* image. See [Custom IRIS-based application image](#custom-iris-based-application-image).

### Versions
Examples in this repository were running on `Ubuntu-22.04` workstation. The following versions were used:

#### Server-side
- Kubernetes 1.24.4
- IKO 3.3.0.120
- Application is based on IRIS 2022.2.0.345

#### Client-side

- [docker](https://docs.docker.com/engine/install/)
```
$ docker version --format='{{.Client.Version}}'
20.10.18
```
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
```
$ kubectl version --client --output=json | jq -r '.clientVersion.gitVersion'
v1.24.4
```
- [helm](https://helm.sh/docs/intro/install/)
```
$ helm version --short
v3.10.0+gce66412
```
- [jq](https://stedolan.github.io/jq/download/)
```
$ jq --version
jq-1.6
```

### Kubernetes cluster
Examples in this repository were running with the following Kubernetes installations:
- Local [k3s](https://k3s.io/) with [k3d](https://k3d.io/) wrapper
- Cloud-based [GKE](https://cloud.google.com/kubernetes-engine)

Choose any other type of Kubernetes cluster installation you like or have an access to.

##### k3s/k3d
[k3s](https://k3s.io) is a lightweight Kubernetes with a lightweight wrapper [k3d](https://k3d.io) that enables us to run local Kubernetes in docker. Installation looks simple:
```
$ curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | TAG=v5.4.6 bash

$ k3d --version
k3d version v5.4.6
k3s version v1.24.4-k3s1 (default)

# Run Kubernetes cluster with one control plane as well as one worker node
$ k3d cluster create my --image rancher/k3s:v1.24.4-k3s1 --agents 1 --servers 1

$ docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Names}}'
CONTAINER ID   IMAGE                            NAMES
9e669c6b5816   ghcr.io/k3d-io/k3d-proxy:5.4.6   k3d-my-serverlb
2e593f13020f   rancher/k3s:v1.24.4-k3s1         k3d-my-agent-0
8e4132053b44   rancher/k3s:v1.24.4-k3s1         k3d-my-server-0

$ kubectl get nodes
NAME              STATUS   ROLES                  AGE   VERSION
k3d-my-agent-0    Ready    <none>                 25s   v1.24.4+k3s1
k3d-my-server-0   Ready    control-plane,master   32s   v1.24.4+k3s1
```
> **Note**: you can remove a local Kubernetes cluster later if necessary:
```
$ k3d cluster list
NAME   SERVERS   AGENTS   LOADBALANCER
my     1/1       1/1      true

$ k3d cluster delete my
```

##### GKE
There are many ways to create a Kubernetes cluster in a Google Cloud. One of the ways is described in [Using Cloud Monitoring to Monitor IRIS-Based Applications Deployed in GKE](https://community.intersystems.com/post/using-cloud-monitoring-monitor-iris-based-applications-deployed-gke) in a section `GKE Creation`.

### Custom IRIS-based application image
Let's deploy a custom IRIS-based application, called [secured-rest-api](https://github.com/intersystems-community/secured-rest-api). Docker image is built in a regular way - find a [Dockerfile](https://github.com/intersystems-community/secured-rest-api/blob/master/Dockerfile), run a build command and then push an image to DockerHub public repository:

```
$ export DOCKERHUB_ACCOUNT=<dockerhub_account> # like 'johndoe'
$ export VERSION=2022.2.0.345
$ git clone https://github.com/intersystems-community/secured-rest-api.git
$ cd secured-rest-api
$ docker build -t secured-rest-api:${VERSION} .
$ docker tag secured-rest-api:${VERSION} ${DOCKERHUB_ACCOUNT}/secured-rest-api:${VERSION}
$ docker login -u ${DOCKERHUB_ACCOUNT} # type DockerHub account password here
$ docker push ${DOCKERHUB_ACCOUNT}/secured-rest-api:${VERSION}
```

### IKO Installation
Let's install IKO and IRIS based application into `iko` namespace.

We will install IKO [Using Helm](https://helm.sh/docs/intro/using_helm/).
```
$ helm repo add intersystems-charts https://charts.demo.community.intersystems.com
$ helm repo update
$ helm search repo intersystems-charts -l
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
intersystems-charts/iris-operator       3.3.0           3.3.0.120       IRIS Operator by InterSystems
intersystems-charts/iris-operator       3.1.0           3.1.0.112       IRIS Operator by InterSystems

# The nodeSelector=null is set here for simplicity to avoid issues with nodes OS-type
$ helm upgrade --install iko          \
    intersystems-charts/iris-operator \
    --version 3.3.0                   \
    --namespace iko                   \
    --create-namespace                \
    --atomic                          \
    --wait                            \
    --set nodeSelector=null

$ helm -n iko ls --all -f iko -o=json | jq '.[]'
{
  "name": "iko",
  "namespace": "iko",
  "revision": "1",
  "updated": "<date>",
  "status": "deployed",
  "chart": "iris-operator-3.3.0",
  "app_version": "3.3.0.120"
}
```

### Minimal application installation
After IKO installation we're ready to install a custom IRIS-based application. IKO can do it for us if we provide a desired cluster definition in yaml file. More details can be found at [Create the IrisCluster definition file](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO#AIKO_clusterdef).

We will start with a minimal viable definition - [01-minimal.yaml](irisclusters/01-minimal.yaml).

> **Note**: don't forget to replace `<dockerhub_account>` by your actual account name.

As this demo application is based on Community IRIS, we don't need a license for it and can use an empty license secret. If an application requires a license, please follow [licenseKeySecret: Provide a secret containing the InterSystems IRIS license key](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO#AIKO_clusterdef_licenseKeySecret) instruction.

So let's create a license secret first (from [empty-iris-license.yaml](empty-iris-license.yaml)):
```
$ kubectl -n iko apply -f empty-iris-license.yaml
```
Then apply an IRIS cluster definition:
```
$ kubectl -n iko apply -f irisclusters/01-minimal.yaml
```
If you take a look at IKO logs you can see it's busy a little creating different Kubernetes resources like Services, Roles, ConfigMaps adn so on:
```
$ kubectl -n iko -l app=iris-operator logs --tail 100
...
I1222 17:19:34.137694       1 iriscluster.go:43] Sync/Add/Update for IrisCluster iko/iris
I1222 17:19:34.167739       1 service.go:37] Creating Service iko/iris-svc.
I1222 17:19:34.191787       1 service.go:37] Creating Service iko/iris.
I1222 17:19:34.227929       1 serviceaccount.go:37] Creating ServiceAccount iko/iris-mirror-labeler.
I1222 17:19:34.239505       1 role.go:37] Creating Role iko/iris-mirror-labeler.
I1222 17:19:34.273744       1 rolebinding.go:37] Creating RoleBinding iko/iris-mirror-labeler.
I1222 17:19:34.305173       1 configmap.go:37] Creating ConfigMap iko/iris-data.
I1222 17:19:34.318595       1 statefulset.go:41] Creating StatefulSet iko/iris-data.
I1222 17:21:02.834895       1 iriscluster.go:43] Sync/Add/Update for IrisCluster iko/iris
I1222 17:21:02.856523       1 service.go:77] Patching Service iko/iris with {"metadata":{"annotations":null}}.
...
```
After creating all these Kubernetes resources, a new single-node IRIS cluster should be available (please wait for *Running* status):
```
$ kubectl -n iko get pods -l intersystems.com/name=iris
NAME          READY   STATUS    RESTARTS   AGE
iris-data-0   1/1     Running   0          6m3s
```

### Check an application availability
Let's leverage a Kubernetes port-forwarding feature:
```
$ kubectl -n iko port-forward svc/iris 52773:52773
```
In another console run a *GET* request:
```
$ curl -sku Bill:ChangeMe http://localhost:52773/crudall/persons/all | jq '.[0]'
{
  "Name": "Pybus,Heloisa T.",
  "Title": "Strategic Resources Director",
  "Company": "PicoSys Partners",
  "Phone": "729-284-4473",
  "DOB": "1923-04-29"
}
```
Run a *POST* request and check a result:
```
$ curl -ku John:ChangeMe -XPOST -H "Content-Type: application/json"  http://localhost:52773/crud/persons/ -d '{"Name":"John Doe"}'

$ curl -sku Bill:ChangeMe http://localhost:52773/crud/persons/all | jq '.[0]'
{
  "Name": "John Doe"
}
```

Let's kill an IRIS application pod to check if data survive pod restarts:
```
$ kubectl -n iko delete pod iris-data-0
```

Wait until it starts again (`Running` status and `Ready 1/1`):
```
$ kubectl -n iko get pod iris-data-0 -w
```

Check if newly created person exists. Stop the previous *post-forward* by pressing *Ctrl + C* and run it again:
```
$ kubectl -n iko port-forward svc/iris 52773:52773

# In another console
$ curl -sku Bill:ChangeMe http://localhost:52773/crud/persons/all | jq '.[0]'
{
  "Name": "John Doe"
}
```
As we see recently created user still exists. Press `Ctrl+C` to stop port-forwarding.

### Application Installation with custom settings
Default settings are good enough to run an IRIS-based application almost on every Kubernetes cluster but you would definitely be interested in ways to change them according to your needs.

Let's examine what IKO could provide in that sense. Ask a question - what and how could be changed? For now, you might guess that it needs something to be changed in *iriscluster* definition file. And you're right. Then you have several options to find out exact settings.

#### Official documentation
[Create the IrisCluster definition file](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO#AIKO_clusterdef) page gives a lot of useful examples. It worth to spend some time there.

#### Kubectl explain
The next stop could be `kubectl explain` usage. Example:
```
$ kubectl explain iriscluster
```
More details could be found by using *dot* notation. For instance, let's find what types of topologies are supported:
```
$ kubectl explain iriscluster.spec.topology
...
   arbiter	<Object>
     Arbiter node of an IRIS cluster.

   compute	<Object>
     Compute nodes of an IRIS cluster.

   data	<Object> -required-
     Data nodes of a IRIS cluster.

   iam	<Object>
     IAM node of an IRIS cluster

   sam	<Object>
     SAM node of an IRIS cluster

   webgateway	<Object>
     WebGateway node of an IRIS cluster.
...
```
Travelling back and forth you might find all available settings and their description.

#### Resources (CPU, memory) setting
If you're not very familiar with resources requests/limits you could spend some time reading [Kubernetes best practices: Resource requests and limits](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-resource-requests-and-limits) and [Understanding Kubernetes limits and requests by example](https://sysdig.com/blog/kubernetes-limits-requests/).

After that let's examine resources defaults first. No CPU/memory resources are set for an application container. It's done specifically to enable IRIS to be running on almost each Kubernetes installation either desktop or cloud-based.
```
$ kubectl -n iko get pods iris-data-0 -ojsonpath='{.spec.containers[*].resources}'
{}
```
Now let's take a look at [02-resources.yaml](irisclusters/02-resources.yaml) where we define requests and limits (arbitrary values) for CPU/memory.

Apply this updated manifest:
```
$ kubectl -n iko apply -f irisclusters/02-resources.yaml
```
The current IRIS pod will be terminated and a new one, with defined resources, is up. If you did follow this readme and didn't make other changes, persistent volumes should be on place during this operation, i.e. data/wij/journals should survive this update.

Let's examine resources after update, they should be equal to what we've set:
```
$ kubectl -n iko get pods iris-data-0 -ojsonpath='{.spec.containers[*].resources}' | jq .
{
  "limits": {
    "cpu": "300m",
    "memory": "1Gi"
  },
  "requests": {
    "cpu": "300m",
    "memory": "1Gi"
  }
}
```

Remember that CPU resources are compressible, i.e. when an application reaches CPU limit, it starts to be throttled but still works though slower. See [CPU limits and aggressive throttling in Kubernetes](https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718) for more details.

Memory resources aren't compressible. If application reaches Memory limit, it can be just killed due to Out Of Memory issue. See [How to fix OOMKilled Kubernetes error (exit code 137)](https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/) for more examples.

[How to troubleshoot Kubernetes OOM and CPU Throttle](https://sysdig.com/blog/troubleshoot-kubernetes-oom/) article also looks good for reading in this area.

Of course, it's better to monitor resources usage to actually define if provided resources are fit application needs. InterSystems recommends usage of [System Alerting and Monitoring](https://docs.intersystems.com/components/csp/docbook/Doc.View.cls?KEY=PAGE_sam) that could be enabled in IKO by providing [sam definition](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO#AIKO_clusterdef_sam).

Another option is usage of OpenSource tools like [Prometheus Operator](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus) for metrics, [Grafana Loki](https://grafana.com/docs/loki/latest/installation/helm/) for logs etc.

Monitoring options are going to be considered with more details in another repository.

#### IRIS Configuration parameters setting
IRIS can be configured with [Configuration Parameter File](https://docs.intersystems.com/csp/docbook/DocBook.UI.Page.cls?KEY=RACS_cpf) (CPF). Some settings are already changed in CPF file by IKO. They can be reviewed by looking at `iris-data` ConfigMap:
```
$ kubectl -n iko get configmap iris-data -ocustom-columns='DATA:.data'
DATA
map[data.cpf:

[Startup]
DefaultPort=1972
[Actions]
ModifyService:Name=%service_ecp,Enabled=1
CreateDatabase:Name=irisdm,Directory=/irissys/data/IRIS/mgr/irisdm
CreateNamespace:Name=IRISCLUSTER,Globals=irisdm,Routines=irisdm

[Journal]
CurrentDirectory=/irissys/journal1/
AlternateDirectory=/irissys/journal2/

[config]
wijdir=/irissys/wij/
MaxServerConn=64
MaxServers=64
]
```
As we see, WIJ and Journals location are overwritten here. The default location is in `/usr/irissys/mgr/` but to achieve persistence during IRIS pod restarts, WIJ and Journals (primary and secondary) are located at external disks provided by Kubernetes Persistent Volumes.

You can read [Introduction to the Configuration Parameter File](https://docs.intersystems.com/csp/docbook/DocBook.UI.Page.cls?KEY=RACS_cpf) about all configuration options available in CPF.

In relation to IKO, as described in [configSource: Create configuration files and provide a config map for them](https://docs.intersystems.com/components/csp/docbook/DocBook.UI.Page.cls?KEY=AIKO#AIKO_clusterdef_configSource) we could define several CPF files, separated for data and compute nodes, or a common CPF for both IRIS instance types. All configuration settings will be merged as described at [Automating Configuration of InterSystems IRIS with Configuration Merge](https://docs.intersystems.com/csp/docbook/DocBook.UI.Page.cls?KEY=ACMF) page.

As an example, let's create a `common.cpf` file setting a Globals Buffer Cache size. In terms of Kubernetes it should be stored as a ConfigMap - [configmap-common-cpf.yaml](configmap-common-cpf.yaml). Let's deploy it:
```
$ kubectl -n iko apply -f configmap-common-cpf.yaml
```
And let's point an IKO to read this new configuration - see [03-common-cpf.yaml](irisclusters/03-common-cpf.yaml) with `spec.configSource` added:
```
$ kubectl -n iko apply -f irisclusters/03-common-cpf.yaml
```
IRIS instance will be recreated. Examining pod logs should show an updated value for Globals Buffer:
```
$ kubectl -n iko logs -f iris-data-0 | grep -i global
[INFO] log: 12/23/21-17:22:27:392 (799) 0 [Generic.Event] Allocated 103MB shared memory: 16MB global buffers, 35MB routine buffers
Allocated 103MB shared memory: 16MB global buffers, 35MB routine buffers
12/23/21-17:22:32:989 (1212) 0 [Generic.Event] Allocated 689MB shared memory: 512MB global buffers, 80MB routine buffers
```

### Conclusion
We've looked at the process of deployment of IRIS-based application into Kubernetes cluster using InterSystems Kubernetes Operator, with default as well as customized settings like container resources or IRIS configuration in cpf file.