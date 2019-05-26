# Notes from the "Managing the Kubernetes API Server and Pods" Pluralsight course

## Using the Kubernetes API
Get information about the current context, ensure you're logged into the correct cluster:  
~~~  
kubectl config get-contexts  
~~~

Change the context if needed:  
~~~  
kubectl config use-context X  
~~~  

Get information about the API Server for the current context:  
~~~  
kubectl cluster-info  
~~~  

Get available resources from the Kubernetes server:  
~~~  
kubectl api-resources | more  
~~~  

Get resources filtered by api group:  
~~~  
kubectl api-resources --api-group=apps  
~~~  

Get information about the configuration settings for a resource:  
~~~  
kubectl explain pod  
~~~  

Explain resource for a specific api version:  
~~~  
kubectl explain deploy --api-version apps/v1beta2  
~~~  

List api versions available in the cluster:  
~~~  
kubectl api-versions | sort | more  
~~~  

Increase verbosity, get more insight between the interaction of the client and the server.
Display requested resource URL. Focus on VERB, API Path and response code:  
~~~  
kubectl get pod X -v 6  
~~~  

Same output as 6, add HTTP Request Headers. Focus on application type, and User-Agent:
~~~  
kubectl get pod X -v 7  
~~~  

Same output as 7, adds Response Headers and truncated Response Body:
~~~  
kubectl get pod X -v 8  
~~~  

Same output as 8, add full Response. Focus on the bottom, look for metadata:
~~~  
kubectl get pod X -v 9  
~~~  

Start kubectl proxy session, this will authenticate use to the API Server.
After you can use curl to submit requests to the API server:
~~~  
kubectl proxy &  
~~~  

A watch on pods will watch on the resourceVersion on api/v1/namespaces/default/Pods and notify on any changes. 
The resourceVersion changes every time the resource changes: 
~~~  
kubectl get pods --watch -v 6 &  
~~~  

We can see kubectl keeps the TCP session open with the server, waiting for data: 
~~~  
netstat -plant | grep kubectl   
~~~  

Accessing logs: 
~~~  
kubectl get logs X  
kubectl get logs X -v 6  
~~~  

API flow: Connection -> authentication -> authorization -> admission Control  
API Versioning: Alpha/Experimental -> Beta/Pre-release -> Stable/General Availability  

**REST API**  
GET: Get the data for a specified resource(s)
POST: Create resource  
DELETE: Delete resource  
PUT: Create or update entire existing resource  
PATCH: Modify the specified fields of a resource
LOG: Retrieve logs from a container in a Pod  
EXEC: Exec a command in a container and get the output  
WATCH: Change notifications on a resource with streaming output  

**Response codes** (many more, but these are the most common)  
Success (2xx): 200 = OK | 201 = Created | 202 = Accepted  
Client Errors (4xx): 401 = Unauthorized | 403 = Access Denied | 404 = Not Found  
Server Errors (5xx): 500 = Internal Server Error  

**Core API (Legacy)**  
http://apiserver:port/api/$VERSION/$RESOURCE_TYPE  
http://apiserver:port/api/$VERSION/namespaces/$NAMESPACE/$RESOURCE_TYPE/$RESOURCE_NAME  

**API Groups**  
http://apiserver:port/apis/$GROUPNAME/$VERSION/$RESOURCE_TYPE  
http://apiserver:port/apis/$GROUPNAME/$VERSION/namespaces/$NAMESPACE/$RESOURCE_TYPE/$RESOURCE_NAME  

## Managing Objects with Labels, Annotations, and Namespaces

**Organize objects in Kubernetes**  
**Namespaces:** When you want to put a boundary around a resource or an object in terms of security, naming or resource allocations.  
**Labels:** When you want to act on an object or group of objects or influence internal Kubernetes operations.  
**Annotation:** Add a bit more information or metadata about an object or resource  

**Namespaces**  
**default:** When you deploy resources and don’t specify a namespace  
**kube-public:** Automatically and is readable by all users in a cluster, commonly used to store shared objects between namespaces, for access across the whole cluster (like config maps).  
**kube-system:** The system pods, like the api server, etcd, controller manager, kube proxy, etc.  
**User defined:** Create imperatively with kubectl or declaratively in a manifest in YAML.  

### Namespaces
Get a list of all the namespaces in the cluster:
~~~  
kubectl get namespaces  
~~~  

Get a list of all the API resources and if they van be in a namespace:
~~~  
kubectl api-resources --namespaced=true | head  
kubectl api-resources --namespaced=false | head  
~~~  

Get state info: 
~~~  
kubectl describe namespaces  
kubectl describe namespaces X  
~~~  

Create namespace:
~~~  
kubectl create namespace X  
~~~  

Create an object within a namespace:
~~~  
kubectl run nginx --image=nginx --namespace X  
~~~  

Query within a namespace:
~~~  
kubectl get pods --namespace=X  
kubectl get all --namespace=X  
~~~  

Delete all resources in a namespace:
~~~  
kubectl delete namespaces X  
~~~  

### Labels  
Add a label to a pod:  
~~~  
kubectl label pod X key=value  
~~~  

Edit a label for a pod:  
~~~  
kubectl label pod X key=updated-value --overwrite  
~~~  

Delete a label:  
~~~  
kubectl label pod X key-  
~~~  

Querying using labels an selectors.  
Works for more than only pods:  
~~~  
kubectl get pods --show-labels  
kubectl get pods --selector key=value  
kubectl get pods -l 'key in (value,value)'  
kubectl get pods -l 'key notin (value,value)'  
kubectl get nodes --show-labels  
~~~  

By changing the pod-template-hash, you can separate a pod from the deployment.  
Kubernetes won't see the pod anymore and create a new one. In the mean time you can debug the pod, for example:  
~~~  
kubectl label pod X pod-template-hash=DEBUG  
~~~  

### Annotations
Annotations can't be used to query/select Pods or other resources.  
Key up to 63 characters, like a label.  
Values can be up to 256KB, unlike labels.  

Add and overwrite an annotation:  
~~~  
kubectl annotate pod X key=value  
kubectl annotate pod X key=value --overwrite  
~~~  

## Running and Managing Pods

Pod lifecycle: Creation -> Running -> Termination  
Container default restart policy is Always. This is the restart policy for the container inside of the Pod.  

Container probes: livenessProbes and readinessProbes:  
**livenessProbe:** Checks the container, on failure the kubelet restarts the container depending on the restart policy. 
**readinessProbe:** Checks container, application won't receive traffic until ready, on failure removes Pod from load balancing or rc controller, prevent users from seeing errors.  

Diagnostic Checks for probes:  
**Exec:** Measures process exit code in the container.  
**tcpSocket:** Checks if we can successfully open a tcp socket on a container.  
**httpGet:** executes a check on an URL on the container and checks the return code between 200 +> and < 400 is successful.  
Responses: Success Failure, Unknown

Get notifications on kubectl events:  
~~~  
kubectl get events --watch &  
~~~  

Exec into single container deployment:  
~~~  
kubectl exec -it X -- /bin/bash  
~~~  

Exec into multi container deployment:  
~~~  
kubectl exec -it X --container X -- /bin/bash  
~~~  

Increase the verbosity:  
~~~  
kubectl -v 6 exec -it X -- /bin/bash  
~~~  

Run process inside a container:  
~~~  
kubectl exec -it X -- /usr/bin/killall X  
~~~  

Access pod's application directly, without a service and also off the pod network:  
~~~  
kubectl port-forward pod X LOCALPORT:CONTAINERPORT  
curl http://localhost:LOCALPORT  
~~~  

Scale up/down a replicaset:  
~~~  
kubectl scale deployment X --replicas=X  
~~~  

Specify grace period time for an application, to overwrite the default 30 SIGTERM to SIGKILL commands from kubernetes:  
~~~  
kubectl delete pod --grace-period=SECONDS  
~~~  

Force delete pod, delete the metadata and everything else immediately:  
~~~  
kubectl delete pod --grace-period=0 --force  
~~~  