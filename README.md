# Chaos-Workflows

Chaos Workflows are a set of actions strung together to achieve desired chaos impact on a Kubernetes cluster. Workflows
are an effective mechanism to simulate real world conditions & gauge application behaviour in an effective manner. 
Take these usecases for example: 


- Most often, failures do not occur as isolated, single instances. There maybe underlying chronic application-specific or 
  deploy-environment induced conditions on the cluster when other failures occur, effectively causing a multi-component/complex 
  failure. Such as, pod failures occurring when other nodes are in sub-optimal or unschedulable states. 

- Running chaos under highly loaded conditions. The parallel actions of a benchmark run on an app deployment and staggered 
  chaos during this run is highly instructive of the performance characteristic & deployment sanity of the application. 

Workflows are also useful in automating a series of pre-conditioning/setup actions necessary to be performed before triggering 
chaos. 

LitmusChaos leverages the popular workflow & GitOps tool [Argo](https://argoproj.github.io/) to achieve this. Argo facilitates
creation of a whole lot of chaos workflow models while being extremely simple & efficient to use. 

This repository hosts predefined workflows based on LitmusChaos experiments you can pick for use, while also the dev/usage docs
that explain the procedure to construct your own chaos workflows.

## Getting Started

The subsequent section explains how to get started with a simple chaos workflow that disrupts (via pod-delete chaos) a multi-replica 
nginx deployment while a load generator generates benchmark traffic against it. The typical usecase for such a chaos workflow as this 
is to be able to observe the extent of degradation in completed requests & request rate (resiliency & perf indicators). Users can play
around the benchmark as well as chaos parameters as part of a detailed experiment to understand application behaviour & fix the 
bugs/deployment issues and arrive at achievable SLAs. 

### Install Argo Workflow Infrastructure

The Argo workflow infra consists of the Argo workflow CRDs, Workflow Controller, associated RBAC & Argo CLI. The steps
shown below installs argo in the standard cluster-wide mode wherein the workflow controller operates on all
namespaces. Ensure that you have the right permission to be able to create the said resources. 

If you would like to run argo with a namespace scope, refer to [this](https://github.com/argoproj/argo/blob/master/manifests/namespace-install.yaml) manifest.  

- Create argo namespace

  ```
  root@demo:~/chaos-workflows# kubectl create ns argo
  namespace/argo created
  ```

- Create the CRDs, workflow controller deployment with associated RBAC

  ```
  root@demo:~/chaos-workflows# kubectl apply -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/install.yaml
  
  customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
  customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
  customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
  customresourcedefinition.apiextensions.k8s.io/workflowtemplates.argoproj.io created
  serviceaccount/argo created
  serviceaccount/argo-server created
  role.rbac.authorization.k8s.io/argo-role created
  clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-admin configured
  clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-edit configured
  clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-view configured
  clusterrole.rbac.authorization.k8s.io/argo-cluster-role configured
  clusterrole.rbac.authorization.k8s.io/argo-server-cluster-role configured
  rolebinding.rbac.authorization.k8s.io/argo-binding created
  clusterrolebinding.rbac.authorization.k8s.io/argo-binding unchanged
  clusterrolebinding.rbac.authorization.k8s.io/argo-server-binding unchanged
  configmap/workflow-controller-configmap created
  service/argo-server created
  service/workflow-controller-metrics created
  deployment.apps/argo-server created
  deployment.apps/workflow-controller created
  ```

- Verify successful creation of argo resources

  ```
  root@demo:~/chaos-workflows# kubectl get crds | grep argo
  
  clusterworkflowtemplates.argoproj.io                         2020-05-15T03:01:31Z
  cronworkflows.argoproj.io                                    2020-05-15T03:01:31Z
  workflows.argoproj.io                                        2020-05-15T03:01:31Z
  workflowtemplates.argoproj.io                                2020-05-15T03:01:31Z
  ``` 
  
  ```
  root@demo:~/chaos-workflows# kubectl api-resources | grep argo
  
  clusterworkflowtemplates          clusterwftmpl,cwft   argoproj.io                              false        ClusterWorkflowTemplate
  cronworkflows                     cronwf,cwf           argoproj.io                              true         CronWorkflow
  workflows                         wf                   argoproj.io                              true         Workflow
  workflowtemplates                 wftmpl               argoproj.io                              true         WorkflowTemplate
  ```

  ```
  root@demo:~/chaos-workflows# kubectl get pods -n argo
  NAME                                   READY   STATUS    RESTARTS   AGE
  
  argo-server-65cbb4874c-cbq2h           0/1     Running   0          12s
  workflow-controller-55bffbdbfd-c4jdf   1/1     Running   0          12s
  ```

- Install the argo CLI on the harness/test machine (where the kubeconfig is available) 

  ```
  root@demo:~# curl -sLO https://github.com/argoproj/argo/releases/download/v2.8.0/argo-linux-amd64
  
  root@demo:~# chmod +x argo-linux-amd64
 
  root@demo:~# mv ./argo-linux-amd64 /usr/local/bin/argo

  root@demo:~# argo version
  argo: v2.8.0
  BuildDate: 2020-05-11T22:55:16Z
  GitCommit: 8f696174746ed01b9bf1941ad03da62d312df641
  GitTreeState: clean
  GitTag: v2.8.0
  GoVersion: go1.13.4
  Compiler: gc
  Platform: linux/amd64

  ```

### Install Litmus Infrastructure

Refer to the LitmusChaos [documentation](https://docs.litmuschaos.io) to get started on installing Litmus infra on your 
Kubernetes clusters. In this example, we will use the [admin mode](https://docs.litmuschaos.io/docs/admin-mode/) of execution where 
all chaos resources will be created in the centralized namespace, litmus. 


### Install a Sample Application: Nginx 

- Install a simple multi-replica stateless nginx deployment with service exposed over nodeport

  ```
  root@demo:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/chaos-workflows/master/App/nginx.yaml
  
  deployment.extensions/nginx created
  ```
  
  ```
  root@demo:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/chaos-workflows/master/App/service.yaml
   
  service/nginx created 
  ```

  You can access this service over `https://<node-ip>:<nodeport>` 

### Create the Argo Access ServiceAccount

- Create the service account and associated RBAC which will be used by the Argo  workflow controller to execute the 
  actions specified in the workflow. In our case, this corresponds to the launch of the nginx benchmark job, creating 
  the chaosengine to trigger the pod-delete chaos action. In our example, we place it in the namespace where the litmus
  chaos resources reside. 

  ```
  root@demo:~# kubectl apply -f https://raw.githubusercontent.com/litmuschaos/chaos-workflows/master/Argo/argo-access.yaml -n litmus

  serviceaccount/argo-chaos created
  clusterrole.rbac.authorization.k8s.io/chaos-cluster-role created
  clusterrolebinding.rbac.authorization.k8s.io/chaos-cluster-role-binding created
  ```

### Create the Pod-Delete ChaosExperiment CR 

- Create the pod-delete chaosexperiment custom resource in litmus namespace. This example makes use of the [chaostoolkit chart](https://github.com/litmuschaos/chaos-charts/tree/master/charts/chaostoolkit) as the means to execute the chaos. 

  ```
  root@demo:~# kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/chaostoolkit/k8-pod-delete/experiment.yaml -n litmus
  chaosexperiment.litmuschaos.io/k8-pod-delete created
  ```

  ```
  root@demo:~# kubectl get chaosexperiments -n litmus
  NAME            AGE
  k8-pod-delete   13s
  ```

### Create the Chaos Workflow 

- Applying the workflow manifest performs the following actions in parallel: 

  - Starts a nginx benchmark job for specified duration (60s)
  - Triggers a random pod-kill of the nginx replicas by creating the chaosengine CR. Cleans up after chaos.


  ```
  root@demo:~# argo submit https://raw.githubusercontent.com/litmuschaos/chaos-workflows/master/Argo/argowf-chaos-admin.yaml -n litmus
  Name:                argowf-chaos-sl2cn
  Namespace:           litmus
  ServiceAccount:      argo-chaos
  Status:              Pending
  Created:             Fri May 15 15:31:45 +0000 (now)
  Parameters:
    appNamespace:      default
    adminModeNamespace: litmus
    appLabel:          nginx
    fileName:          pod-app-kill-count.json  
  ``` 
  
### Visualize the Chaos Workflow 

- You can visualize the progress of the chaos workflow via the Argo UI. Convert the argo-server service to type NodePort & 
  view the dashboard at `https://<node-ip>:<nodeport>`

  ```
  root@demo:~# kubectl patch svc argo-server -n argo -p '{"spec": {"type": "NodePort"}}'
  service/argo-server patched
  ```



