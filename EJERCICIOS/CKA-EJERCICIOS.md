### Ejercicios de Preparación para la CKA.

- Create a new ClusterRole named demo-clusterrole in demo-namespace. Any resource associated with the cluster role should be able to create the following resources: Deployment, Statefulset, Daemonset.

Create a ServiceAccount named demo-token and bind the ClusterRole with the ServiceAccount.

- ```kubectl create ns demo-ns```
- ```kubectl create clusterrole demo-clusterrole --verb=create --resource=deployment,statefulset,daemonset -n demo-namespace --dry-run=client -o yaml > demo-clusterrole.yaml```
- ```kubectl apply -f demo-clusterrole.yaml -n demo-namespace```
- ```kubectl create serviceaccount demo-token -n demo-namespace```
- ```kubectl create clusterrolebinding demo-clusterrolebinding --clusterrole=demo-clusterrole --serviceaccount=demo-namespace:demo-token --dry-run=client -o yaml > demo-clusterrolebinding.yaml```
- ```kubectl apply -f demo-clusterrolebinding.yaml -n demo-namespace```


- Create three pods, pod name and their image name is given below:


  * nginx	nginx
  * busybox1	busybox1, sleep 3600
  * busybox2	busybox2, sleep 1800

Make sure only busybox1 pod should be able to communicate with nginx pod on port 80. Pod busybox2 should not be able to connect to pod Nginx.

- ```kubectl run nginx --image=nginx ```
- ```kubectl run busybox1 --image busybox1 - sleep 3600 --dry-run=client -o yaml > busybox1.yaml```
- ```kubectl apply -f busybox1.yaml```
- ```kubectl run busybox2 --image busybox2 - sleep 1800 --dry-run=client -o yaml > busybox2.yaml ```
- ```kubectl apply -f busybox2.yaml```

 ```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: nginx
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              run: busybox1
      ports:
        - protocol: TCP
          port: 80
```

- Create a pod “cka-pod” using image “nginx”, with a hard limit of 0.5 CPU and 20 Mi memory in “cka-exam” namespace.

- ```kubectl create namespace cka-exam```
- ```kubectl run cka-pod --image=nginx --dry-run=client -o yaml > cka-pod.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: cka-pod
  name: cka-pod
spec:
  containers:
  - image: nginx
    name: cka-pod
    resources:
      limits:
        cpu: "0.5"
        memory: "20Mi"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- ```kubectl apply -f cka-pod.yaml -n cka-exam```

- Create a pod by the name “read-onlypod” using the image “alpine”. The process running inside the pod should only have ReadOnly access to the container’s filesystem. 

Create a volume by the name “my-volume” and mount it at /data ( inside the container). The process running inside the container should have read & write access on the mounted volume. Also, run “sleep 3600” inside the container.

- ```kubectl run read-onlypod --image=alpine --dry-run=client -o yaml > read-onlypod.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: read-onlypod
spec:
  containers:
  - image: alpine
    name: read-onlypod
    command:
        - "sleep"
        - "3600"
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - mountPath: /data
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: #This guarantees read & write access.
      medium: Memory
      sizeLimit: 500Mi
```
- ```kubectl apply -f read-onlypod.yaml```


- Create a pod by the name “pod-add-settime-capability”, using the image “alpine”. Add capabilities called “SYS_TIME” and “NET_ADMIN” to the container and command “sleep 3600”.

Set system time by executing the command “sudo date +%T –set “16:00:00”.

Once the pod is up and running, check system time and store the output under /tmp/cka/pod-add-settime-capability.txt

- ```kubectl run pod-add-settime-capability --image=alpine --dry-run=client -o yaml > pod-add-settime-capability.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-add-settime-capability
  name: pod-add-settime-capability
spec:
  containers:
  - image: alpine
    name: pod-add-settime-capability
    command: 
     - "sleep"
     - "3600"  
    securityContext:
     capabilities:
      add: ["NET_ADMIN", "SYS_TIME"] 
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- ```kubectl -n default exec -it pod-add-settime-capability -- date```
- ```kubectl -n default exec -it pod-add-settime-capability -- date +%T –set "16:00:00"```
- ```echo "kubectl exec -it pod-add-settime-capability -- date" > /tmp/cka/pod-add-settime-capability.txt```


- Create a multi-pod container by name “multi-container-pod” with below-mentioned details:

##### First container

name: main
Image name: busybox:1.28
args: – /bin/sh – -c – > i=0; while true; do echo “$i: $(date)” >> /var/log/cka-exam.log; i=$((i+1));
 sleep 1; done
volumeMount: /var/log
volumeName: main-volume

##### Second container

name: sidecar
Image name: busybox:1.28
args: [/bin/sh, -c, ‘tail -n+1 -F /var/log/cka-exam.log’]
volumeMount: /var/log
volumeName: main-volume

check the sidecar container’s log and store it’s output under /tmp/output.txt

- ```kubectl run multi-container-pod --image=busybox:1.28 --dry-run=client -o yaml > multi-container-pod.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-container-pod
  name: multi-container-pod
spec:
  containers:
  - image: busybox:1.28
    name: main
    args: [ "/bin/sh","-c", "i=0; while true; do echo “$i: $(date)” >> /var/log/cka-exam.log; i=$((i+1)); sleep 1; done"]  
    volumeMounts:
    - name: main-volume
      mountPath: /var/log  
  - image: busybox:1.28
    name: sidecar
    args: ["/bin/sh", "-c", "tail -n+1 -F /var/log/cka-exam.log"]
    volumeMounts:
    - name: main-volume
      mountPath: /var/log    
    resources: {}
  volumes:
    - name: main-volume    
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
- ```kubectl exec -it multi-container-pod -c sidecar -- sh```
- ```cp -rf /var/log/cka-exam.log > /tmp/output.txt```
- ```cat /tmp/output.txt```


- Create a Pod called non-root-pod , image: redis:alpine

  * runAsUser: 1000
  * fsGroup: 2000

- ```kubectl run non-root --image=redis:alpine --dry-run=client -o yaml > non-root.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: non-root
  name: non-root
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000  
  containers:
  - image: redis:alpine
    name: non-root
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
- ```kubectl apply -f non-root.yaml```


- You have got a project which has three running pods by below given names:

Pod Name	Image	Namespace
WebServer	devopstitan/cka-exam:webserver-p80	web
appserver	devopstitan/cka-exam:appserver-p8080	app
dbserver	devopstitan/cka-exam:dbserver-p6379	db

Create a network policy by name “cka-networkpolicy” to allow only one way connection  from Webserver pod to  appserver and DB server . 

The appserver is running on the TCP port 8080 and dbserver is running on the TCP port 6379

- ```kubectl create ns web```
- ```kubectl create ns app```
- ```kubectl create ns db```
- ```kubectl run webserver --image=devopstitan/cka-exam:webserver-p80 -n web```
- ```kubectl run appserver --image=devopstitan/cka-exam:appserver-p8080 -n app```
- ```kubectl run dbserver --image=devopstitan/cka-exam:dbserver-p6379 -n db```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cka-networkpolicy
  namespace: web
spec:
  podSelector:
    matchLabels:
      run: webserver
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: app
        - podSelector:
            matchLabels:
              run: app
      ports:
        - protocol: TCP
          port: 8080
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: db
        - podSelector:
            matchLabels:
              run: db
      ports:
        - protocol: TCP
          port: 6379
```

- ```kubectl apply -f cka-networkpolicy.yaml```

- As a DevOps engineer, you are asked to create a MySQL deployment by the name cka-deployment, and to fulfill the project requirement you also need to create the following Kubernetes objects:

  *  Create a secret by name “cka-secret”  for storing the database password.
  MYSQL_ROOT_PASSWORD=ckapassword

  * Create a Persistent Volume by name “cka-pv” and allocate storage space of 500 Mi.

  * Create a Persistent Volume Claim by “cka-pvc” of size 100Mi, make sure cka-pvc is bound to cka-pv.

  * Create deployment with below mentioned details: 
    Name: cka-deployment
    Image: devopstitan/cka-exam:mysql-question-09
    Secret: cka-secreh
    Port:  3306
    PVC: cka-pvc
    mountPath: /var/lib/mysql

  * Create a service by the name “cka-service” and expose MYSQL port 3306

- ```kubectl create secret generic cka-secret --from-literal=MYSQL_ROOT_PASSWORD=ckapassword```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cka-pv
spec:
  storageClassName: manual
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
  claimRef:
    name: cka-pvc
    namespace: default
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cka-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: cka-deployment
  name: cka-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cka-deployment
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cka-deployment
    spec:
      containers:
      - image: devopstitan/cka-exam:mysql-question-09
        name: cka-exam
        volumeMounts:
        - name: task-pv-storage
          mountPath: "/var/lib/mysql"
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: cka-secret
              key: password    
        ports:
          - containerPort: 3306
            name: "http-server"    
        resources: {}
      volumes:
      - name: task-pv-storage
        persistentVolumeClaim:
          claimName: cka-pvc        
status: {}
```

- ``` kubectl expose deployment cka-deployment --port=3306 --target-port=3306 --name=cka-service```


- Switch to context “kubernetes-admin@kubernetes” and Create a deployment by name cka-deployment, as per the below requirements:

Replicas: 4
Image : nginx:1.14
namespace: cka-exam
Implement strategy for rolling update with maxUnavailable: 50%

 
  * Update deployment : Image: nginx:1.15

  * Update Deployment once again: Image: nginx:1.99

Check the rollout status & rollout history.
After 2nd rollout, if pods are not up then roll back the deployment to its initial version (image=nginx:1.14)

- ``` kubectl config use-context kubernetes-admin@kubernetes```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
  namespace: cka-exam
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14
        name: nginx
        resources: {}
status: {}
```
- ``` kubectl -n cka-exam set image deployment/nginx nginx=nginx:1.15```
- ```kubectl -n cka-exam set image deployment/nginx nginx=nginx:1.99```
- ``` kubectl -n cka-exam rollout status deployment/nginx```
- ```kubectl -n cka-exam rollout history deployment/nginx```
- ```kubectl -n cka-exam rollout undo deployment/nginx --to-revision=1```

- List pods Sorted by Restart Count. 
List PersistentVolumes sorted by capacity. 
List Services Sorted by Name

- ``` kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'```
- ``` kubectl get pv --sort-by=.spec.capacity.storage```
- ``` kubectl get services --sort-by=.metadata.name```

- Create a new service account with the name pvviewer. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropriate cluster role called pvviewer-role and ClusterRoleBinding called pvviewer-role-binding.
Next, create a pod called pvviewer with the image: redis and serviceaccount: pvviewer in the default namespace.

- ```kubectl create serviceaccount pvviewer```
- ```kubectl create clusterrole pvviewerrole --verb=list --resource=persistentvolumes```
- ``` kubectl create clusterrolebinding pvviewer-role-binding --clusterrole=pvviewer --serviceaccount=default:pvviewer```

- ```kubectl run pvviewer --image=redis --dry-run=client -o yaml > pvviewer-pod.yaml```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pvviewer
  name: pvviewer
  namespace: default  
spec:
  containers:
  - image: redis
    name: pvviewer
    resources: {}
  serviceAccountName: pvviewer    
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
- ```kubectl apply -f pvviewer-pod.yaml ```

- Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Record the version. Next upgrade the deployment to version 1.17 using rolling update. Make sure that the version upgrade is recorded in the resource annotation.

- ``` kubectl create deployment nginx-deploy --image=nginx:1.16 --replicas=1```
- ``` kubectl set image deployment/nginx-deploy nginx=nginx:1.17```
- ```Annotations: deployment.kubernetes.io/revision: 2```

- Create snapshot of the etcd running at https://127.0.0.1:2379. Save snapshot into /opt/etcd-snapshot.db.

- ``` ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save  /opt/etcd-snapshot.db```

- Create a Persistent Volume with the given specification.
Volume Name: pv-analytics, Storage: 100Mi, Access modes: ReadWriteMany, Host Path: /pv/data-analytics

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/pv/data-analytics"
```

- ```kubectl apply -f pv-analytics.yaml```

- Taint the worker node to be Unschedulable. Once done, create a pod called dev-redis, image redis:alpine to ensure workloads are not scheduled to this worker node. Finally, create a new pod called prod-redis and image redis:alpine with toleration to be scheduled on node01.
key:env_type, value:production, operator: Equal and effect:NoSchedule

- ```kubectl taint node node01 env_type=production:NoSchedule```
- ```kubectl run prod-redis --image=redis:alpine --dry-run=client -o yaml > prod-redis.yaml ```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: prod-redis
  name: prod-redis
spec:
  containers:
  - image: redis:alpine
    name: prod-redis
    resources: {}
  tolerations:
  - key: "env_type"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule" 
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

- Set the node named worker node as unavailable and reschedule all the pods running on it. (Drain node)

- ```kubectl drain node <worker node> --ignore-daemonsets```

- Create a Pod called non-root-pod , image: redis:alpine
  - runAsUser: 1000
  - fsGroup: 2000

- ```kubectl run non-root-pod --image=redis:alpine --dry-run=client -o yaml > non-root-pod.yaml``` 

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: non-root-pod
  name: non-root-pod
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - image: redis:alpine
    name: non-root-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Create a NetworkPolicy which denies all ingress traffic.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

-  Create a pod myapp-pod and that use an initContainer that uses the busybox image and
sleeps for 20 seconds.

- ```kubectl run myapp-pod --image=nginx --dry-run=client -o yaml > myapp.pod```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', "sleep 20"]

```


- Create the ingress resource with name ingress-wear-watch to make the applications available
at /wear on the Ingress service in app-space namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear-watch.com
    http:
      paths:
      - pathType: Prefix
        path: "/wear"
        backend:
          service:
            name: wear-service
            port:
              number: 8080
```


## UDEMY COURSE MOCK EXAMS (CKA COURSE)

- Deploy a pod named nginx-pod using the nginx:alpine image.
- ```kubectl run nginx-pod --image=nginx:alpine```

- Deploy a messaging pod using the redis:alpine image with the labels set to tier=msg.
- ```kubectl run messsaging --image=redis:alpine --labels="tier=msg"```

- Create a namespace named apx-x9984574.
- ```kubectl create ns apx-x9984574```

- <p style="color:red;">Get the list of nodes in JSON format and store it in a file at /opt/outputs/nodes-z344kd9.json</p>
- ```kubectl get nodes -o json > /opt/outputs/nodes-z344kd9.json```

- Create a service messaging-service to expose the messaging application within the cluster on port 6379.
- ```kubectl expose pod messaging --port=6379 --name=messaging-service```

- Create a deployment named hr-web-app using the image kodekloud/webapp-color with 2 replicas.
- ```kubectl create deployment hr-web-app --image=kodekloud/webapp-color --replicas=2```

- <p style="color:red;">Crate a static pod named static-busybox on the controlplane node that uses the busybox image and the command sleep 1000.</p>
- ```kubectl run static-busybox --image=busybox --dry-run=client -o yaml --command --sleep 1000 > static-busybox.yaml```
- ```mv static-busybox.yaml > /etc/kubernetes/manifests```

- kubectl create a pod in the finance namespace named temp-bus with the image redis:alpine.

- ```kubectl run temp-bus --image=redis:alpine --namespace=finance```

- Expose the hr-web-app as service hr-web-app-service application on port 30082 on the nodes on the cluster. The web application listens on port 8080.
- ```kubectl expose deploy hr-web-app --name=hr-web-app-service --type=NodePort --port 8080```
- ```kubectl edit svc hr-web-app-service```

- Use JSON PATH query to retrive the osImage of all the nodes and store it in a file /opt/outputs/nodes_os_x43kj56.txt The osImage  are under the nodeInfo section under status of each node.
- ```kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.osImage}'```

- Take a backup of the etcd cluster and save it to /opt/etcd-backup.db
- ```ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/etcd-backup.db```

- <p style="color:red;">Crate a new user called john. Grant him access to the cluster. John should have permission to create, list, get, update and delete pods in the development namespace. The private key exists in the location: /root/CKA/john.key and csr at /root/CKA/john.car</p>

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbTlvYmpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0NnZ0VCQUt2Um1tQ0h2ZjBrTHNldlF3aWVKSzcrVVdRck04ZGtkdzkyYUJTdG1uUVNhMGFPCjV3c3cwbVZyNkNjcEJFRmVreHk5NUVydkgyTHhqQTNiSHVsTVVub2ZkUU9rbjYra1NNY2o3TzdWYlBld2k2OEIKa3JoM2prRFNuZGFvV1NPWXBKOFg1WUZ5c2ZvNUpxby82YU92czFGcEc3bm5SMG1JYWpySTlNVVFEdTVncGw4bgpjakY0TG4vQ3NEb3o3QXNadEgwcVpwc0dXYVpURTBKOWNrQmswZWhiV2tMeDJUK3pEYzlmaDVIMjZsSE4zbHM4CktiSlRuSnY3WDFsNndCeTN5WUFUSXRNclpUR28wZ2c1QS9uREZ4SXdHcXNlMTdLZDRaa1k3RDJIZ3R4UytkMEMKMTNBeHNVdzQyWVZ6ZzhkYXJzVGRMZzcxQ2NaanRxdS9YSmlyQmxVQ0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQ1VKTnNMelBKczB2czlGTTVpUzJ0akMyaVYvdXptcmwxTGNUTStsbXpSODNsS09uL0NoMTZlClNLNHplRlFtbGF0c0hCOGZBU2ZhQnRaOUJ2UnVlMUZnbHk1b2VuTk5LaW9FMnc3TUx1a0oyODBWRWFxUjN2SSsKNzRiNnduNkhYclJsYVhaM25VMTFQVTlsT3RBSGxQeDNYVWpCVk5QaGhlUlBmR3p3TTRselZuQW5mNm96bEtxSgpvT3RORStlZ2FYWDdvc3BvZmdWZWVqc25Yd0RjZ05pSFFTbDgzSkljUCtjOVBHMDJtNyt0NmpJU3VoRllTVjZtCmlqblNucHBKZWhFUGxPMkFNcmJzU0VpaFB1N294Wm9iZDFtdWF4bWtVa0NoSzZLeGV0RjVEdWhRMi80NEMvSDIKOWk1bnpMMlRST3RndGRJZjAveUF5N05COHlOY3FPR0QKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  usages:
  - digital signature
  - key encipherment
  - client auth
```

- ```cat john.csr | base64 | tr -d "\n"```
- ```kubectl apply -f john-csr.yaml```
- ```kubectl certificate approve john-developer```
- ```kubectl create role developer --namespace=development --verbs=create,get,list,update,delete --resource=pods```
- ```kubectl create rolebinding john-developer --role=developer --user=john -n development```
- ```kubectl auth can-i get pods --namespace=development --as john```


- Create a nginx pod called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service.

- ```kubectl run nginx-resolver --image=nginx```
- ```kubectl expose pod nginx-resolver --name=nginx-resolver-service --port=80```
- ```kubectl run busybox --image=busybox:1.28 -- sleep 4000```
- ```kubectl exec busybox -- nslookup nignx-resolver-service > /root/CKA/nginx.svc```
- ```kubectl exec busybox -- nslookup nignx-resolver-pod > /root/CKA/nginx.pod```

- Create a static pod on node01 called nginx-critical with image nginx and make sure that it is recreated/restarted automatically in case of failure. Use /etc/kubernetes/manifests as the Static Pod path for example.

- ```kubectl run nginx-critical --image=nginx --dry-run=client -o yaml > static.yaml```
- ```scp static.yaml node01:/root/```
- ```ssh node01```
- ```mkdir -p /etc/kubernetes/manifests```
- ```vi /var/lib/kubelet/config.yaml (Add staticPodPath=/etc/kubernetes/manifests)```
- ```cp /root/static.yaml /etc/kubernetes/manifests/```
- ```exit```
- ```kubectl get pods```

- Create a new service account with the name pvviewer. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropiate cluster role called pvviewer-role and ClusterRoleBinding called pvviewer-role-binding. Next, create a pod called pvviewer with the image: redis and serviceAccount: pvviewer in the default namespace.

- ```kubectl create serviceaccount pvviewer```
- ```kubectl create clusterrole pvviewer-role --verb=list --resource=persistentvolume```
- ```kubectl create clusterrolebinding pvviewer-role-binding --clusterole=pvviewer-role --serviceaccount=default:pvviewer```

- List the InteralIP of all nodes of the cluster. Save the result to a file /root/CKA/node_ips. Answer shoud be in the format: InternalIP of controlplane InternalIp of node01 (in a single line).

- ```kubectl get nodes -o json | jq | grep -i internalIP -B 100```
- ```kubectl get nodes -o json | jq -c 'paths'```
- ```kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address} > /root/CKA/node_ips```








