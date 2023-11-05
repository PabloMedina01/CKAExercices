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




