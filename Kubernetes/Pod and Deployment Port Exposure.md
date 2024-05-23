
This page is to explain the process of portforwarding and proxy etc. Basically from scratch to full deployments. 


Firstly, let's start with just a generic pod.
`kubectl run nginx --image=nginx`

Let's double check that this is working correctly.
`kubectl get pods -o wide`

![[Pasted image 20240523113951.png]]

Cool, so we have our pod running. Let's just test the networking connection. My Node is running on `192.168.0.249`. We can also check this by using `kubectl get nodes -o wide`.

![[Pasted image 20240523114247.png]]

Let's expose that pod and try a `curl` command. 
`kubectl port-forward pod/nginx 8080:80`

![[Pasted image 20240523114922.png]]

In my case, I have to open a new SSH window, to keep the `port-forward` alive. Then as below, use `curl localhost:8080`.

![[Pasted image 20240523115009.png]]

So we can see that the access is allowed from the `localhost`. But this is where I started getting confused. We need to utilise the `--address` parameter in order to change what address to listen on. When we ran the first `port-forward` command, we only exposed it to the `localhost`. Hence the `127.0.0.1` address. 

So let's cancel out of the original `port-forward` on our first SSH window, using `CTRL+C` and let's run another command.

We'll use `kubectl port-forward --address 192.168.0.249 pod/nginx 8080:80` this time and let's see the results.
![[Pasted image 20240523115411.png]]

Note that the output mentions the IP Address and is no longer `localhost`. Now if we try from another PC, or device on the LAN, we can access it by browsing to `192.168.0.249:8080`

![[Pasted image 20240523115537.png]]

Great! We now have access to our pod from outside of the Node. But what do we do when we're trying to keep this as a permanent solution? We can't have an SSH window open with the `port-forward` command running forever. That's not very stable. So let's cancel out of the current `port-forward` command and see what else we can do, again, using `CTRL+C` to break out of it. 

First of all, we need to change our approach. Let's start with a deployment and expose that to our host. For ease, we will use the below `YAML` file.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

Take note under the `spec` section, or more specifically `replicas: 1`.  This means that the deployment will have two pods created, if we were to change this to 10, it would create 11 pods. Also, you can find more info [here](https://kubernetes.io/docs/tutorials/services/connect-applications-service/). I've saved the `YAML` as `run-my-nginx.yaml`.

So let's deploy this by using `kubectl apply -f ./run-my-nginx.yaml`, and then let's view our deployment by using `kubectl get deployment/my-nginx`.

![[Pasted image 20240523135652.png]]

The pods are available, and in a Ready state. Let's try and see if we can can use `curl` to get the data from the landing page. We'll use `kubectl get pods -o wide` again to find the ip address of the pods.

![[Pasted image 20240523140115.png]]

Let's try the `curl` command now. 

![[Pasted image 20240523140226.png]]

So, we were able to access the pods this time without port-forward, why is that? Well, a deployment is different to just deploying a new pod. Let's quickly cover this to enhance our understanding.

### Pods

- **Direct Access:** When you deploy a standalone pod in Kubernetes, it gets an internal IP address assigned by the cluster. This IP is typically only accessible within the cluster network.
- **Port-Forwarding:** If you need to access this pod from outside the cluster (e.g., from your local machine), you have to use `kubectl port-forward`. This command creates a tunnel between your local machine and the pod, forwarding traffic from a local port to a port on the pod.

### Deployments

- **Managed Pods:** A deployment is a higher-level abstraction that manages a set of identical pods (replicas) to ensure availability and scalability.
- **Services:** Deployments are typically paired with a Kubernetes Service. A Service is an abstraction that defines a logical set of pods and a policy by which to access them. Services provide a stable endpoint (IP and port) that you can use to access the pods managed by the deployment.

So to summarize, we don't need to use `port-forward` with deployments because the pods will be given a static IP and port that can be accessed from the node. However, on my main pc, let me try browse to the host IP with the port and see what happens.

![[Pasted image 20240523140927.png]]

We get a failure message. So whilst the cluster has access to the pods, that's because it's opened using the `ClusterIP` service. Which means it can be accessed from within the node and that's it. And to add to that, here is a snippet from the official docs explaining the `ClusterIP` service.

`Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default that is used if you don't explicitly specify a type for a Service.`

Overall we were able to reach the pods individually, from the Node. But before we cover external access, let's first see how we can access the deployment via a single IP. But to also check this, we'll need to make a change in the `HTML` to see what node we are hitting. I personally feel the result would be more satisfying.

So let's use `kubectl get pods -o wide` again to view the names of our pods in the deployment, in my case, the first pod has a name of `my-nginx-684dd4dcd4-tbr4z`. Make sure you write down the IP of the pod too!

I'll use the next command, `kubectl exec -it my-nginx-684dd4dcd4-tbr4z -- bash` to create an interactive window, inside the pod and execute `bash`. The `--` is the parameter separator that Kubernetes recognises. 

![[Pasted image 20240523143410.png]]

As you can see, we're now inside the pod. Let's browse to the location of the html that nginx uses. Typically it is found in `/usr/share/nginx/html`. Once in the folder, we'll see the `index.html` page. 

Let's edit this to display the IP address of the pod we're currently inside. I'll use `nano` for this, because we're not doing anything intense. However, first we'll need to install it so let's use `apt update && apt install nano`, I added a part to the `<p>` section with the IP address - `The IP of this Pod is - 10.42.0.32!`. We will save changes by using `CTRL+X` and then `y` to save changes and `enter` to finish saving. To clean up, we'll just remove `nano` for the sake of space and best practices,`apt remove nano` and finally we'll quit the pod by just typing `exit`. 

Now we'll just double check our service IP address again by using `kubectl get svc`, in my case the IP is `10.43.225.30` so I'll use the command `curl 10.43.225.300` and see the results.

The first attempt hit the second pod;
![[Pasted image 20240523145314.png]]

We can tell because our changes to the `HTML` are not present. Let's try it again and see what happens.

![[Pasted image 20240523145344.png]]

Great! We hit the second pod. However, it's worth noting that this isn't exactly the best use case for load balancing. But it works for our internal setup. If you were using Azure or AWS, then you'd need to look into the service `LoadBalancer` and go from there. That's a bit out of scope for our current situation however, although we will cover a type of `LoadBalancer` next!

In this next step, we need to rewind a couple bits. We can utilise the same deployment as before, and keep our HTML change too, to help us identify we're in a good state. Let's clean up the service by using `kubectl delete svc my-nginx`. We then need to install a new tool called `MetalLB`. I got the below commands from [here](https://metallb.universe.tf/installation/) under the section 'Installation by Manifest'. 

`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml`

We'll create a new `YAML` file, called `metallb-config.yaml` either by using `touch`, or creating it with your favourite IDE/Text-Editor. And once inside we'll use the below. Bear in mind, that you may want to change your range below, the addresses I've listed here I know are free.

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.0.50-192.168.0.110
```

We'll then apply this config, by using `kubectl apply -f metallb-config.yaml`

After completing the above steps, we can use the below to disable the original `servicelb` in order to use `MetalLB` as our `LoadBalancer`. 

`curl -sfL https://get.k3s.io | sh -s - --disable servicelb`

We then need to change the configuration to use an address pool that we specify. You can change this on the last line.

```
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - local-pool

---

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: local-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.50-192.168.0.110
EOF
```

After that's applied, we can finally expose our deployment.

`kubectl expose deploy my-nginx --port 80 --type=LoadBalancer`

Last step, check the IP by using `kubectl get svc` and look for your service, and then under the external IP you should have what you need. So we can now access it from our browser with LoadBalancing enabled!

For the very end of this, I'll assist with some cleanup. Mostly because I had an issue with the service not deleting correctly. It hung indefinitely and I believe that's because when we expose the deployment, it still references it as a standard kubernetes LoadBalancer. There seems to be a cleanup step that is added there but I didn't find any left over issues with this so let's get to it!

Delete the deployment using: `kubectl delete deployment/my-nginx`
Delete the service using: `kubectl delete service/my-nginx`

The below step may be required if MetalLB wasn't fully configured before applying the service. Though I personally thought it finished, it didn't and so this step will help you clean up if not.

**If you have trouble deleting the service.**
Then we need to make a change to the nginx service. Use `kubectl edit service/my-nginx` and delete the line that says 'Finalizer'. This edit tool uses VI so you will need to save using `:wq!`. Once done, we can delete the service. 

Run a quick `kubectl get all` to check if anything was left behind.

And there we have it. A quick run through regarding network access for pods and deployments within a Kubernetes local homelab. 