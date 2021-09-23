# Minikube
# 1.Requirements
```bash
apt install -y socat
apt install -y  apt-transport-https
apt install -y conntrack
apt install -y  virtualbox virtualbox-ext-pack
####Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
install minikube-linux-amd64 /usr/local/bin/minikube
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
cp kubectl /usr/local/bin/
chmod +x  /usr/local/bin/kubectl
```
In my case I set up these aliases (~/.bashrc) and autocompletion:
```bash
cat <<EOF >> ~/.bashrc
source <(kubectl completion bash)
source <(minikube completion bash)
alias k=kubectl
alias kga='watch -x kubectl get all -o wide'
EOF
```

# 2.Start
```bash
minikube start --vm-driver=none
systemctl enable kubelet.service
```
# Check out
```bash
k cluster-info
kubectl get po -A 
OR
minikube kubectl -- get po -A
```
# 3.Creating a sample user
```bash
touch dashboard-adminuser.yaml
```
then fill it as below:
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
# 4.Creating a ClusterRoleBinding
```bash
touch rolebinding.yaml 
```
then fill it as below:
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
# 5.Your Token
to catchig your secret key run this command:
```bash
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```
here you go copy the token and paste it into Enter token field on the login screen.
# 6.Dashboard
```bash
minikube addons list
minikube addons enable dashboard
minikube dashboard --url
```
TOKEN FOR DASHBOARD: 
```bash
kubectl -n kube-system describe secret deployment-controller-token
```
# 7.Visit dashboard on local
You need to setting up Tunnel to your VMware so run this command with your details:
```bash
ssh -i D:\YOURKEY.ppk -L 8081:localhost:PORTOFYOURDASHBOARD root@VMIP
```
"Do not close your ssh session"
Now visit your dashboard on your system (your link will looks like mine)
```bash
http://127.0.0.1:8081/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```
Here you go ;)

# All about node in minikube
# Node wasn't ready:
In my case I set up these aliases (~/.bashrc) and autocompletion:
```bash
cat <<EOF >> ~/.bashrc
source <(kubectl completion bash)
source <(minikube completion bash)
alias k=kubectl
alias kga='watch -x kubectl get all -o wide'
EOF
```
If your Node was not ready check these steps:
```bash
kubectl get pods -n kube-system        #ALL system pods must be ready
```
###### Check which node wasn't in ready state:
```bash
    kubectl get nodes 
    kubectl describe node nodename #nodename which is not in readystate.
    kubectl describe node minikube-m03
```
###### ssh to the node:
```bash
   minikube ssh -n minikube-m03
   systemctl status kubelet #Make sure kubelet is running.
   systemctl status docker #Make sure docker service is running.
   sudo journalctl -u kubelet # To Check logs in depth.
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
```
In case you still didn't get the root cause, check below things:
Make sure your node has enough space and memory. Check for /var directory space especially. 
command to check: 
```bash
free -m
```
Verify cpu utilization with top command. and make sure any process is not taking an unexpected memory.

# Node Labeling:
###### How to set Label to Nodes:
```bash
k label node minikube-m02 demo=true
k get node minikube-m02 --show-labels
k describe nodes minikube-m02 | grep demo
``` 
###### Use Labels on your Deployment:
Add this line to your deployment file:
```bash
  nodeSelector:
    demo: "true"   
```
Sample:

```bash
git clone https://github.com/RFinland/kubenode.git
cd kubenode
k create -f 1-Using-Labels.yaml	
```
You can see what was you set for your pod(s):
```bash
k get pods
#In my case one of my pods named asnginx-deploy-799b7f9cc6-2t4mr
k describe pod/nginx-deploy-799b7f9cc6-2t4mr  | grep Selectors
```
If you change scale of replicas, all new pods deployed only on you labeled node that you mention on you deployment yaml file:
```bash
k scale deploy nginx-deploy --replicas=4   ####It'll be run a new pod on minikube-m02
```
###### Remove your label
```bash
kubectl label node minikube-m02 demo-
```
###### Add more than one label:
```bash
k label node minikube-m02 demo=true proxy=enable 
```
###### Remove your labels
```bash
k label node minikube-m02 demo- proxy-
```
# Deploy with labels on diff namespaces

###### 1.First of all Actvie PodNodeSelector option:
```bash
minikube ssh -n minikube
```
Edit kube-apiserver.yaml to add PodNodeSelector option:
```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```
Then add PodNodeSelector to enable-admission-plugins:
```bash
   - --enable-admission-plugins=...    to ===>    - --enable-admission-plugins=PodNodeSelector,....
```
###### 2. Next We have to set our labels to nodes:
```bash
k label node minikube-m02 env=prod
k label node minikube-m03 env=dev
```
kube-apiserver-minikube will be reastart to watch:
```bash
k -n kube-system get pods
```

###### 3. Now create NameSpaces:
Create NameSpaces with these commands:
```bash
k create ns prod
k create ns dev
k get ns
```
###### 4. Edit NameSpace:
Edit NameSpace config to use specific label:
```bash
k edit ns dev
```
then type
```bash
:syntax off
```
Add this line under name: dev  :
```bash
  annotations:
    scheduler.alpha.kubernetes.io/node-selector: "env=dev"
```	
source:(https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
With this setting we told to our namespace only use deployments with dev label to nodes that have dev label	

######  5. Deploy your pod
In this case I deployed a sample nginx:
```bash
k -n dev  create deploy nginx --image nginx
k -n dev scale deployment.apps/nginx --replicas=4
```
Now We see what we want to see, our deployment is deployed on node who has dev Label, in my case minikube-m03
```bash
k -n dev get pods -o wide
```
ShortSteps:
First NameSpace(Just play with label X  ,k edit ns dev ) Then Node(labeled X ,k label node minikube-m03 env=dev) last Deployment (Deploy On ns contains our label)Our Pods created on Node who labeled X

# DaemeoSets
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

Some typical uses of a DaemonSet are:
- running a cluster storage daemon on every node
- running a logs collection daemon on every node
- running a node monitoring daemon on every node

In a simple case, one DaemonSet, covering all nodes, would be used for each type of daemon. A more complex setup might use multiple DaemonSets for a single type of daemon, but with different flags and/or different memory and cpu requests for different hardware types.
```bash
k apply -f 2-Using-Daemonset.yaml
k get ds
```
First it will be set demotype=nginx-daemonset-demo to all of the nodes then Deployed on all of them. 
Then If you add a new node :
```bash
minikube node add
```
Atleast your new node gets ready (a copy of a Pod runs as well) :
```bash
kga 
```

```bash
#Watch list of ds system:
k get daemonset -n kube-system
#Describe your daemonset:
k describe daemonset nginx-daemonset
k describe pod/nginx-daemonset-ldbjz
```

If you delete youe daemonset:
```bash
k delete daemonset.apps/nginx-daemonset 
```
All of your pods will be delete
Also you can use nodeSelector on your yaml file of daemonse (think about it)

# JOB
A Job creates one or more Pods and will continue to retry execution of the Pods until a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created. Suspending a Job will delete its active Pods until the Job is resumed again.

A simple case is to create one Job object in order to reliably run one Pod to completion. The Job object will start a new Pod if the first Pod fails or is deleted (for example due to a node hardware failure or a node reboot).

You can also use a Job to run multiple Pods in parallel
```bash
kubectl create -f 3-Job-Echo.yaml
```
Then run : 
```bash
kubectl logs pod/helloworld-jzznp
```
You will see the Hello Kubernetes!!! as Completed job, You can see Job completed wih theese commands:
```bash
k get jobs
k describe job helloworld 
```
Till now we created a job to run this command : command: ["echo", "Hello Kubernetes!!!"]
You can do what you need.
If you delete your job , created pod will be deleted clearly
```bash
k delete job helloworld
kga
```
Another sample: run a job contains Sleep:
- completions: 2 means run twice (creating 2 pods) sequential 
- parallellism: 2 means create these two pods at same time 

```bash
kubectl create -f 4-Job-Sleep.yaml
```
After sleep 60 secs mean job done as well and state will be Completed.
Another sample: If your command fails it return Error and create pod again and again and ...
so we can set limit for our fails:
- backoffLimit: 2 means if you achive 2 times fails do not create antoher pod to do your job.
```bash
k create -f 5-Job-FailsLimit.yaml
```
Another sample: Next with activeDeadlineSeconds we set life time to our job:
- activeDeadlineSeconds: 10 
```bash
k create -f 6-Job-DeadlineSec.yaml
```
Your pod will terminate at 10 secs (age 10s) and delete
# CronJob
In my case I set to  schedule: "* * * * *" means ru this job every minutes
For more https://en.wikipedia.org/wiki/Cron and other helpful website: https://crontab.cronhub.io/
```bash
k create -f 7-Cronjob.yaml
```
###### List of Cronjobs:
```bash
k get cronjobs
k get cronjob
k get cj
```
###### Delete whole jobs:
```bash
k delete cj helloworld-cron
k delete -f 7-Cronjob.yaml
```
###### Describe cronjob to see SuccessfulCreate and SawCompletedJob :
```bash
k describe cronjob helloworld-cron
```

###### successfulJobsHistoryLimit and failedJobsHistoryLimit
minikube will be create 3 jobs  and keep them alive , every minutes that job done, itll create another job and pod and delete oldest pod and job that successed 
Because Successful Job History Limit is 3 and Failed Job History Limit is 1 (k describe cronjob helloworld-cron)
To change these valuse we can set "successfulJobsHistoryLimit: 0" it means dont keeep successful pods of your job and "failedJobsHistoryLimit: 0" dont keep failed ones
```bash
k create -f 8-Cronjob-ChangeDefaults.yaml
kga
```
So as you see after job successed pod will be delete.
######  Suspend
To set suspend (Pause) to our cronjob add "suspend: true"
```bash
k create -f 9-Cronjob-Suspend.yaml
kga
```
As you see SUSPEND value is True. OR you can change suspend of your job (Or any other options) commandly:
```bash
k patch cronjob helloworld-cron -p '{"spec":{"suspend": false}}'
```
Other samples (Notice: do not forget to put a space before your values such as :[space]0)
```bash
k patch cronjob helloworld-cron -p '{"sepc":{"successfulJobsHistoryLimit": 0}}'
k patch cronjob helloworld-cron -p '{"spec":{"schedule": "* * * * *"}}'
```
If you see "cronjob.batch/helloworld-cron patched" that means you did it sompletly well and your cronjob altered.
###### concurrencyPolicy
If you set concurrencyPolicy: Allow     #its default  or /Forbid / Replace
when jobs failed or done what happen next, keep old job, replae it, or forbid to create new one 
```bash
k create -f 10-Cronjob-concurrencyPolicy.yaml
```
When you use Replace (Active jobs is 1 New job will Replaced with new)when you got 4 pods (in this case failed pods) your Old job deleted and new job Replaced with it and old pods replaced too.
When you use Forbibd (Active jobs is 1 Dont Create new Job at all)when you got nonstop pods (in this case failed pods) your Old job Remains and new job didnt create.
When you use Allow   (Active jobs get upper)when you got 4 pods (in this case failed pods) your Old job Remains and new job will create also new pods.

######  TTLAfterFinished
if you want to delete job after finish automatically add this feature to your api and manager :
```bash
minikube ssh -n minikube
apt update
apt install nano 
nano /etc/kubernetes/manifests/kube-apiserver.yaml
```
add :
```bash
- --feature-gates= TTLAfterFinished=true
```
also add it to kube-controller-manager.yaml
```bash
nano /etc/kubernetes/manifests/kube-controller-manager.yaml
```
 Your pod/kube-apiserver-minikube and  pod/kube-controller-manager-minikube will be restart:
 ```bash
 kga -n kube-system 
 ```
(look at the age of these pods)

Then you can use ttlSecondsAfterFinished option in your job file:
```bash
k create -f 11-Job-AutoDelete.yaml
```

# Init Container
Init containers: specialized containers that run before app containers in a Pod. Init containers can contain utilities or setup scripts not present in an app image.
###### Understanding init containers
A Pod can have multiple containers running apps within it, but it can also have one or more init containers, which are run before the app containers are started.
Init containers are exactly like regular containers, except:
    Init containers always run to completion.
    Each init container must complete successfully before the next one starts.
If a Pods init container fails, the kubelet repeatedly restarts that init container until it succeeds. However, if the Pod has a restartPolicy of Never, and an init container fails during startup of that Pod, Kubernetes treats the overall Pod as failed.
Fro more detail: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
In my case I defined volume and changed default index.html of nginx via initcontainer busybox :
```bash
k create -f 12-init-container.yaml
k describe deploy nginx-deploy
```
Now expose your deployment:
If you set Port forward n your  editor or on your system you  can achieve to your deployment with these commands 
your forwarded port must be 8080 , thats it.
```bash
k expose deploy nginx-deploy --type NodePort --port 80
kubectl port-forward service/nginx-deploy 8080:80
```
So you can visit http://localhost:8080/ on your local machine.
You can scale up you deployments as below:
```bash
k scale deploy nginx-deploy --replicas=2
```
Now lets have a failed initcontainer:
```bash
k create -f 13-init-container-Failed.yaml
```
Now run these command to see whats heppened:
```bash
kga 
```
See the pod status and Restart count, also your deployment not ready and will not ready, lets describe 
```bash
k describe pod
#OR
k describe pod | grep State,Reason
#OR
k describe pod | grep Reason
```
and see Events lists 
```bash
k describe deployment 
```
You can see your command : "/dani" that ruined everything.

# PV-hostPath
First of all we will create our pv also We set our path inner pv yaml file:
```bash
 hostPath:
    path: "/kube"
```
this path will create on your node in feature
```bash
k create -f 14-pv-hostpath.yaml
k get pv
```
Now create your pvc:
```bash
k create -f 14-pvc-hostpath.yaml
k get pv,pvc 
```
Your pvc STATUS is Bound and Volume is  pv-hostpath and on your pv you can see CLAIM value is default/pvc-hostpath
Befor creating pvc these values was: on your pv CLAIM : empty  and STATUS : Available
```bash
k describe pvc pvc-hostpath
#You can see your hostpath directory
k describe pv pv-hostpath
```
To delete your  pvc you can run:
```bash
k delete pvc pvc-hostpath
```
Now lets create a pod:
in define of our pod we set  persistentVolumeClaim: to claimName: pvc-hostpath and mountPath: /mydata ( inner pod)
with these options we use pvc and linked it to  /mydata on our pod:
```bash
k create -f 14-busybox-pv-hostpath.yaml
```
now go to container and create a file :
```bash
k exec busybox ls      ##### k exec -it busybox -- sh
```
You can see that /mydata path created as well.
Now touch a file:
```bash
k exec busybox touch /mydata/hello
```
now see which node used for your pod the ssh to it:
```bash
minikube ssh -n minikube-m02
ls /kube     #----hello is there
```
Lets delete our pod:
```bash
k delete pod busybox
```
pv and pvc will be exists 
```bash
k get pv,pvc
```
lets delete PVC :
```bash
k delete pvc pvc-hostpath
```
Now ssh again to our node:
```bash
minikube ssh -n minikube-m02
 ls /kube      #----hello exists
```
lets delete PV :
```bash
k delete pv pv-hostpath
minikube ssh -n minikube-m02
 ls /kube      #----hello exists 
```
As you saw /kube/hello doesnt delete.
We will set label to our "another" node via this command: 
```bash
k label node minikube-m03 demoserver=true
```
If you select wrong node to set label delete it via this command :
```bash
kubectl label node minikube-m02 demoserver-
```
Lets take a lokk to ur labeled node:
```bash
k get nodes -l demoserver=true
```
Now add nodeSelector to our pod yaml file:
```bash
added nodeSelector:
        demoserver: "true"
```
Now take a look to busybox container to see hello is there or not
```bash
k exec busybox ls /mydata/
```
There is no hello inner busybox 
same inner minikube-m03 that our pod deployed on it. but /kube directory created . 
Now lets touch something:
```bash
k exec busybox touch /mydata/something
k exec busybox ls /mydata/
minikube ssh -n minikube-m03
ls /kube  #something is there
```
exit and test it to another node:
```bash
minikube ssh -n minikube-m02
ls /kube  #Just old hello was there
```
Now delete pod and pv,pvc to take an action next step:
```bash
k delete pod --all
k delete pv,pvc --all
```
###### Change retain to DELETE:
Add persistentVolumeReclaimPolicy: Delete to your pv , I did in 14-pv-hostpath-RetainDelete.yaml
lets create pv and pvc :
```bash
k create -f  14-pv-hostpath-RetainDelete.yaml
k get pv 
# you can see RECLAIM POLICY value is Delete (not Retain)
k create -f 14-pvc-hostpath.yaml
k get pv,pvc 
```
If you delete pvc with RECLAIM POLICY set delete, our pv have to delete , but it doesnt, our pv remains:
```bash
k delete pvc --all
```
Just  STATUS of pv changed to Failed lets take alook why:
```bash
k describe pv  #OR k describe pv-hostpath
```
you can see this message on Events section:
Warning  VolumeFailedDelete  persistentvolume-controller  host_path delete only supports /tmp/.+ but received provided /kube
So if you chang /kube path to /tmp/kube , whn you delete pvc , your pv will be delete as well.
 
# Secrets

A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in a container image. Using a Secret means that you don't need to include confidential data in your application code.

Because Secrets can be created independently of the Pods that use them, there is less risk of the Secret (and its data) being exposed during the workflow of creating, viewing, and editing Pods. Kubernetes, and applications that run in your cluster, can also take additional precautions with Secrets, such as avoiding writing confidential data to nonvolatile storage.

Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.
```bash
echo -n 'myusername' | base64
echo -n 'mypassword' | base64
```
Th output:
```bash
bXl1c2VybmFtZQ==
bXlwYXNzd29yZA==
```
Fill those value inner your yaml file:
```bash
nano  15-secrets.yaml
```
Then create your secret:
```bash
k create -f 15-secrets.yaml
k get secrets
k get secret secret-demo -o yaml
```
######  (--from-literal):
```bash
k create secret generic secret-demo2 --from-literal=username=myusername --from-literal=password=mypassword
k get secret secret-demo2 -o yaml
```
You can see other options: 
```bash
k create secrets --help
```

######  (--from-file):
```bash
vi username 
#fill it with your value
vi password
#fill it with your value
```
then 
```bash
k create secret generic secret-demo3 --from-file=./username --from-file=./password
```

######  As an Environment
It happens as a  Volume or as an Environment, Ensure your secret secret-demo was exists:
Create a pod:
```bash
k create -f  15-pod-secret-env.yaml
k exec -it busybox -- sh
env | grep myusername
echo $myusername
```

######  As a Volume:
```bash
k create -f 15-pod-secret-volume.yaml
k exec -it busybox -- sh
cd mydata/
```
password & username are there.


# ConfigMaps
A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.
List of configmaps:
```bash
k get cm
k get configmaps
k describe configmap kube-root-ca.crt
k describe cm kube-root-ca.crt
```
lets create a cm demo-configmap:
```bash
k create configmap demo-configmap --from-literal=project.name=omega --from-literal=DSNurl=http://v7.sentry.cc
```
###### Using ConfigMap as an Env:
Now create a Pod with env that valueFrom configMapKeyRef:
```bash
k create -f  16-pod-configmap-env.yaml
k exec -it busybox -- sh
env | grep PROJECT
```
###### Using ConfigMap as an Volume:
Now create pod and use config map as a Volume:
If your DATABASE config was somewhere (in my case /home/minikube/misc/my.cnf)
```bash
k create cm mysql-demo-config --from-file=/home/minikube/misc/my.cnf  
k get cm mysql-demo-config -o yaml       #you can see your /home/minikube/misc/my.cnf detail
```
then create a pod :
```bash
k create -f  6-pod-configmap-mysql-volume.yaml
k exec -it busybox sh 
ls /mydata #my.cnf is there
```
If you edit your cm inner of your busybox /mydaya/my.cnf will be change aftr a while also it happens when you use cm as an env:
```bash
k edit cm mysql-demo-config
k exec -it busybox sh 
cat /mydata/my.cnf
```

###### immutable 
Notice: if you add this option you can't change secret / config map anymore 
```bash
k edit cm demo-configmap
```
Upper of metadata add :
```bash
immutable: true 
```
Save it and exit.
after set immutable: true you can not edit your cm :
 data: Forbidden: field is immutable when `immutable` is set
Also you can not delete this option to edit your cm or secret
You have to delete your cm or secret to able to edit them.

# ResourceQuota
When several users or teams share a cluster with a fixed number of nodes, there is a concern that one team could use more than its fair share of resources.

Resource quotas are a tool for administrators to address this concern.

A resource quota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per namespace. It can limit the quantity of objects that can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by resources in that namespace.

First lets play with context:
```bash
#List of contexts:
kubectl config get-contexts   #context who has * is your current context
#current context
k config current-context
#Create a context linked to your namespace:
k create ns quota-demo-ns
k config set-context quota-demo-ns --namespace=quota-demo-ns --user=minikube --cluster=minikube
#Change current context:
k config use-context quota-demo-ns
```
#Delete The contexts (if you want to recreate or ...):
```bash
kubectl config get-contexts  
kubectl config unset clusters.quota-demo-ns
kubectl config unset users.quota-demo-ns
kubectl config unset contexts.quota-demo-ns
```

Now Create quota:
```bash
k create -f 17-quota-count.yaml
```
In this config we set on namespace: quota-demo-ns our limits are pods: "2"  and configmaps: "2"
```bash
#List of quotas:
k get quota
#Make sure your current-context is quota-demo-ns or run this command:
k config use-context quota-demo-ns
#describe created quota:
k describe quota quota-demo
#OR
k -n quota-demo-ns describe quota quota-demo
```
Now lets play with configmap limits , in our quota yaml file we set configmaps: "2" :
```bash
k create cm cm1 --from-literal=name=testme
k describe quota quota-demo
```
You can not create another configmap:
```bash
k create cm cm2 --from-literal=name=testme
```
It returns :
```
error: failed to create configmap: configmaps "cm2" is forbidden: exceeded quota: quota-demo, requested: configmaps=1, used: configmaps=2, limited: configmaps=2
```
Also you can test it with pod, you can not creat more than two pods.
```bash
k create deploy nginx --image nginx 
k scale deploy nginx  --replicas=2
```
But you can not do:
```bash
k scale deploy nginx  --replicas=3
kga
```
You can see DESIRED =3 but CURRENT = 2 on replicasets section, and in deployment section: READY 2/3 
if you describe deployment:
```bash
k describe deployment.apps/nginx
# Replicas: 3 desired | 2 updated | 2 total | 2 available | 1 unavailable
```
Till now we limit cm and pods , also you can limit job,cron job and mem or cpu 

###### Limit MEM:
```bash
k delete quota quota-demo
k create -f 17-quota-mem.yaml
k get quota
```

Lets create a Pod:
```bash
k create -f 17-pod-quota-mem.yaml
```
So :
```bash
Error from server (Forbidden): error when creating "17-pod-quota-mem.yaml": pods "nginx" is forbidden: failed quota: quota-demo-mem: must specify limits.memory
```
Beacuse you have to set limits in your pod definition:
```bash
lets define limits:
	resources:
	  limits:
		memory: "100Mi"
```
So
```bash
k create -f 17-pod-quota-mem-limits.yaml
```
Now pod created.
```bash
k get quota
#You can see limits.memory: 100Mi/500Mi
```

Also you can define limit range for all , and no need to define any single pods definitis.
```bash
k delete quota quota-demo-mem
k create -f 17-quota-limitrange.yaml
k get limitrange
k describe pod/nginx
```
You can see what you set in limitrange :
```bash
 Limits:
      memory:  300Mi
    Requests:
      memory:     50Mi
```	  
So 
```bash
k create -f  17-reqlimits.yaml
k get quota
```

You will see:
```bash
  REQUEST                    LIMIT
 requests.memory: 0/100Mi   limits.memory: 0/500Mi
```
Now lets create a pod:
```bash
k create -f  17-limitresources.yaml
```
Error :
```bash
Error from server (Forbidden): error when creating "17-limitresources.yaml": pods "nginx" is forbidden: exceeded quota: quota-demo-mem, requested: requests.memory=200Mi, used: requests.memory=0, limited: requests.memory=100Mi
```
it means you have to define requests limit as below:
```bash
k create -f  17-limitresources-fixed.yaml
kga
```
As you have seen in this exercise, you can use a ResourceQuota to restrict the memory request total for all Containers running in a namespace. You can also restrict the totals for memory limit, cpu request, and cpu limit.

# NFS
Network File Sharing (NFS) is a protocol that allows you to share directories and files with other Linux clients over a network. Shared directories are typically created on a file server, running the NFS server component. Users add files to them, which are then shared with other users who have access to the folder.

An NFS file share is mounted on a client machine, making it available just like folders the user created locally. NFS is particularly useful when disk space is limited and you need to exchange public data between client computers.

###### On your NFS SERVER:
```bash
apt -y install nfs-kernel-server
#OR 
apt install nfs-server
systemctl enable nfs-server
systemctl start  nfs-server
vi /etc/exports
#add the path
/srv/nfs/kubedata     *(rw,sync,no_subtree_check,insecure)
```
Create the path:
```bash
mkdir -p /srv/nfs/kubedata
chmod -R 777 /srv/nfs
```
See settings:
```bash
exportfs -rav
#exporting *:/srv/nfs/kubedata
exportfs -v
showmount -e
#/srv/nfs/kubedata *
```
Test your node before do it with minikube:
###### ON YOUR NODE worker:
```bash
minikube ssh -n minikube-m02
ping 185.97.118.212
showmount -e 185.97.118.212   #IP of your nfsserver
sudo -i
mount -t nfs 185.97.118.212:/srv/nfs/kubedata /mnt
mount | grep kubedata
umount /mnt
```
Everything is good, lets play with minikube:
###### minikube
Set your nfs ip inner 18-pv-nfs.yaml and run:
```bash
k create -f 18-pv-nfs.yaml
k get pv
k create -f  18-pvc-nfs.yaml
k get pvc
k get pv,pvc
```
Now create and deploy your pod:
```bash
k create -f  18-nfs-nginx.yaml
k get pods
k expose deploy nginx-deploy --type NodePort --port 80
kubectl port-forward service/nginx-deploy 8080:80
```
visit http://localhost:8080/
Change your index.html:
You can change your index.html on your nfsserver :
```bash
cd  /srv/nfs/kubedata
nano /srv/nfs/kubedata/index.html
```
change it as you want then just refresh your browser

# Statefulsets
StatefulSet is the workload API object used to manage stateful applications.

Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.

Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

If you want to use storage volumes to provide persistence for your workload, you can use a StatefulSet as part of the solution. Although individual Pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new Pods that replace any that have failed.
Using StatefulSets

StatefulSets are valuable for applications that require one or more of the following.

    Stable, unique network identifiers.
    Stable, persistent storage.
    Ordered, graceful deployment and scaling.
    Ordered, automated rolling updates.

In the above, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn't require any stable identifiers or ordered deployment, deletion, or scaling, you should deploy your application using a workload object that provides a set of stateless replicas. Deployment or ReplicaSet may be better suited to your stateless needs.
Limitations

    The storage for a given Pod must either be provisioned by a PersistentVolume Provisioner based on the requested storage class, or pre-provisioned by an admin.
    Deleting and/or scaling a StatefulSet down will not delete the volumes associated with the StatefulSet. This is done to ensure data safety, which is generally more valuable than an automatic purge of all related StatefulSet resources.
    StatefulSets currently require a Headless Service to be responsible for the network identity of the Pods. You are responsible for creating this Service.
    StatefulSets do not provide any guarantees on the termination of pods when a StatefulSet is deleted. To achieve ordered and graceful termination of the pods in the StatefulSet, it is possible to scale the StatefulSet down to 0 prior to deletion.
    When using Rolling Updates with the default Pod Management Policy (OrderedReady), it's possible to get into a broken state that requires manual intervention to repair.
On NFS Server:
```bash
mkdir /srv/nfs/kubedata/{pv0,pv1,pv2,pv3,pv4}
sudo chmod -R 777 /srv/nfs
```
On your host:

nano 19-sts-pv.yaml and set your nfsserver ip :
```bash
nano 19-sts-pv.yaml
```
then:
```bash
k create  -f 19-sts-pv.yaml
```
Then create your pod:
```bash
k create  -f 19-sts-nginx.yaml
```
On your nfsserver go to the path of your used pv (to  findout which pv file):
```bash
k get pv 
#see the bounded pvs and  claimed to default/www-nginx-sts-1  
#OR
k get pvc 
#www-nginx-sts-1   Bound    pv-nfs-pv2 
```
So on your nfsserver go to this path :
```bash
cd /srv/nfs/kubedata/pv2
touch hello
```
For sure:
```bash
k exec -it pod/nginx-sts-1 /bin/bash
cd /var/www
ls 
```
You can see hello in this path .
Now delete your pod:
```bash
k delete pod nginx-sts-1
```
It will delete the pod (nginx-sts-1) and recreate it again:
Then on your nfsserver:
```bash
ls /srv/nfs/kubedata/pv2
```
and ur file exists , all pod use same pvc 
#and ur file exists , all pod use same pvc 
```bash
k get sts
```

Lets delete everything:
```bash
k delete sts nginx-sts
k delete service/nginx-headless
```
Other way:
```bash
k scale sts nginx-sts --replicas=0
```
Also your pvc and pv remains, you haveto delete them mannually :
```bash
k delete pvc --all
k delete pv --all
```
Now lets do it parralel :
19-sts-nginx-Parralel.yaml
```bash

k create  -f 19-sts-nginx-Parralel.yaml
kga
```
As you see status is Pending and you have to create your pv (your pvc will be create automatically):
```bash
k create  -f 19-sts-pv.yaml
kga
```
Notice after delete everything you have to check pvc list that which one connected to your pod.

# Dynamically provision NFS persistent volumes with Helm
Install the Helm:
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

helm repo list
```
Add a repo to Helm:
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update 
```
lets see list of clusterrole,clusterrolebinding,role,rolebinding  who is contain nfs:
```bash
k get clusterrole,clusterrolebinding,role,rolebinding | grep nfs   
```
There is no role,rolebinding ,clusterrole, clusterrolebinding contains nfs
So to install the chart with the deploy name nfs-client, run the following command:
 #its creating serviceaccount,clusterrole,clusterrolebinding,role and rolebinding
 ```bash
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=185.97.118.212 \
    --set nfs.path=/srv/nfs/kubedata
```
Now:
```bash	
kga
 k describe pod/nfs-subdir-external-provisioner-75bc8dd749-kj5v6 | less
```
You will see:
```bash
 Environment:
      PROVISIONER_NAME:  cluster.local/nfs-subdir-external-provisioner
      NFS_SERVER:        185.97.118.212
      NFS_PATH:          /srv/nfs/kubedata
    Mounts:
      /persistentvolumes from nfs-subdir-external-provisioner-root (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-9cs6g (ro)
```
We didnt have pv and pvc lets test it now and create PVC,  first replace storageclass name inner pvc yaml file:
```bash
k get storageclass
 #OR
k get sc
```
Now create a pvc: 
```bash
k create -f 20-pvc-nfs-sc.yaml
```
PV automatically created 
```bash
k get pv,pvc
```
on your nfsserver :
```bash
ls /srv/nfs/kubedata
```

# Horizontal Pod Autoscaler or (HPA) CPU
First ,We have to enable metrics-server addons
```bash
minikube addons list
minikube addons enable metrics-server
#to disable addon:
minikube addons disable  metrics-server
```
then : 
```bash
minikube start   #maybe its not nesseccary
k top pods -n kube-system
k top nodes -n kube-system
k top pods 
k top nodes
k get services -n kube-system   
#OR 
kubectl get svc -n kube-system 
#You will see the metrics-server as a service 
k get pods -n kube-system
#You will see the metrics-server-77c99ccb96-htgtc as a pod
k get deployments -n kube-system
#You will see the metrics-server  as a deployment
kubectl logs -n kube-system deploy/metrics-server
#Logs of metrics-server deployment 
```
Lets create our deployment :
```bash
nano 21-php-apache.yaml
```
in the container section you see:
```bash
        resources:
          limits:
            cpu: 100m
          requests:
            cpu: 100m
#these lines are our metric-server (limits)
```
then:
```bash
kubectl apply -f 21-php-apache.yaml
```
Then define your HorizontalPodAutoscaler(HPA):
We set maxReplicas to 5 and minReplicas to  1 and   targetCPUUtilizationPercentage to 20%
```bash
k create -f 21-hpa.yaml
#OR use this commnd: 
kubectl autoscale deployment php-apache --cpu-percent=20 --min=1 --max=5
kga
```
Now lets create some trafic inner pod:
```bash
k exec -it pod/php-apache-5b7bcbc4f4-ng49f -- sh
```
You can run "yes" or "apt update" or ...
open new termianl and run 
```bash
kga
```
you will see minikube create new pods to handle trafic, and max pods was 5.

Now lets terminate yes/apt update  command with press ctrl+c , then you will see new pods will be deleted automatically after less more 7 minutes.
# Horizontal Pod Autoscaler or (HPA) Memory
Make sure  metrics-server addon is enabled.
```bash
minikube addons enable metrics-server
```
Lets create our deployment :
```bash
nano 22-nginx-mem.yaml
```
in the container section you see:
```bash
        resources:
          limits:
            memory: 1Gi
          requests:
            memory: 0.5Gi
```
#these lines are our metric-server (limits)
```bash
k create -f 22-nginx-mem.yaml
```
Then define your HorizontalPodAutoscaler(HPA):
```bash
k create -f 22-hpa-mem.yaml
```
Then lets make some trafic:
```bash
k exec -it pod/php-apache-6856c856bd-g9mgk  -- sh
```
inside container: 
```bash
apt update
apt install stress
stress --vm 2 --vm-bytes 200M 
```
out of your pod, run:
```bash
kga
```
as you see new pods created, after you terminate stress , new pods will terminate after 6-7 minutes.
# Disruptions

This guide is for application owners who want to build highly available applications, and thus need to understand what types of disruptions can happen to Pods.

It is also for cluster administrators who want to perform automated cluster actions, like upgrading and autoscaling clusters.
Voluntary and involuntary disruptions

Pods do not disappear until someone (a person or a controller) destroys them, or there is an unavoidable hardware or system software error.

We call these unavoidable cases involuntary disruptions to an application. Examples are:

    a hardware failure of the physical machine backing the node
    cluster administrator deletes VM (instance) by mistake
    cloud provider or hypervisor failure makes VM disappear
    a kernel panic
    the node disappears from the cluster due to cluster network partition
    eviction of a pod due to the node being out-of-resources.

Except for the out-of-resources condition, all these conditions should be familiar to most users; they are not specific to Kubernetes.

We call other cases voluntary disruptions. These include both actions initiated by the application owner and those initiated by a Cluster Administrator. Typical application owner actions include:

    deleting the deployment or other controller that manages the pod
    updating a deployments pod template causing a restart
    directly deleting a pod (e.g. by accident)

Cluster administrator actions include:

    Draining a node for repair or upgrade.
    Draining a node from a cluster to scale the cluster down (learn about Cluster Autoscaling ).
    Removing a pod from a node to permit something else to fit on that node.

These actions might be taken directly by the cluster administrator, or by automation run by the cluster administrator, or by your cluster hosting provider.

Ask your cluster administrator or consult your cloud provider or distribution documentation to determine if any sources of voluntary disruptions are enabled for your cluster. If none are enabled, you can skip creating Pod Disruption Budgets.

    Caution: Not all voluntary disruptions are constrained by Pod Disruption Budgets. For example, deleting deployments or pods bypasses Pod Disruption Budgets.

First create your dployment:
```bash
k create deploy nginx --image nginx
#Scale up to 4
k scale deploy nginx --replicas=4
#Define our Pod Disruption Budget:
k create pdb pdbdemo --min-available 50% --selector "app=nginx"
#OR you can use yaml file : k create -f 23-pdb.yaml
#list of PDB:
k get pdb
#Describe of PDB
k describe pdb pdbdemo
```
Now if you drain a node :
```bash
k drain minikube-m03 --ignore-daemonsets
#( to undrain : kubectl uncordon minikube-m03)
```
You will see your pod deployed on another node, and stayed 4 as you declare in replicas=4
```bash
 #delete all of your pdbs:
 k delete pdb  --all
 ```
 
 

# VerticalPodAutoscaler(VPA)
Make sure metrics-server addon is enabled.
```bash
minikube addons enable metrics-server
```
List of VPAs: 
```bash
k get vpa
```

Then we need to autoscalar:
```bash
git clone https://github.com/RFinland/autoscaler.git
cd ~/kubenode/autoscaler/vertical-pod-autoscaler
```
run:
If you already have a autoscaler run :
```bash
./hack/vpa-down.sh
#OR to create:
./hack/vpa-up.sh
```
To print YAML contents with all resources that would be understood by kubectl diff|apply|... commands, you can use
```bash
./hack/vpa-process-yamls.sh print
```
Lets see some change after install autoscaler:
```bash
k get pods -n kube-system
```
As you see:
vpa-admission-controller
vpa-recommender
vpa-updater
Added as well.

Then :
#We set update : Off
Lets create VerticalPodAutoscaler(VPA):
```bash
k create -f  24-vpa-updateModeOff.yaml
k get vpa
k describe vpa my-vpa
```
```bash

nano 24-VPA-Deployment-Labeled.yaml
```
As you can see We have to set selector so we have to set label to our nodes :
```bash
k label node minikube-m02 app=my-app-deployment
k label node minikube-m03 app=my-app-deployment
```
Then 
```bash
k create -f  24-VPA-Deployment-Labeled.yaml
kga
k get vpa 
```
When PROVIDED setted to True then run :
```bash
k describe vpa my-vpa
```
You will see Recommendation:
```bash
  Recommendation:
    Container Recommendations:
      Container Name:  my-container
      Lower Bound:
        Cpu:     25m
        Memory:  262144k
      Target:
        Cpu:     25m
        Memory:  262144k
      Uncapped Target:
        Cpu:     25m
        Memory:  262144k
      Upper Bound:
        Cpu:     95051m
        Memory:  99371500k
```
 Lets set update : Auto , so 
 ```bash
 k delete deployment --all 
 k delete vpa my-vpa  
```
Now we will create a deployment with limit resources :
```bash
k create -f  24-VPA-Deployment-Labeled-Limits.yaml
k describe pod/my-app-deployment-6b9df99b4-fpb5h | less
```
As you can see there: 
```bash
    Requests:
      cpu:        100m
      memory:     50Mi
```
Now lets create VPA with update Auto :
```bash
k create -f  24-vpa-updateModeAuto.yaml
```
after a while, pods recreated automatically, lets describe our new pods and VPA:
```bash
k describe vpa my-vpa
```
#      Target: Cpu:     587m and  Memory:  262144k will set to pods
```bash
k describe pod/my-app-deployment-6b9df99b4-4jrgr | less
```
As you can see:
```bash
    Requests:
      cpu:        548m
      memory:     262144k
```

Here you go ;)

