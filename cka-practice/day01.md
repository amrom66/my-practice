1.列出环境内所有的pv 并以 name字段排序（使用kubectl自带排序功能）
```shell
kubectl get pv --sort-by=.metadata.name
```

2.列出指定pod的日志中状态为Error的行，并记录在指定的文件上

```code
kubectl logs <podname> | grep bash > /opt/KUCC000xxx/KUCC000xxx.txt
```

3.列出k8s可用的节点，不包含不可调度的 和 NoReachable的节点，并把数字写入到文件里

```code
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}'  && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"
```

4.创建一个pod名称为nginx，并将其调度到节点为 disk=stat上
```code
kubectl run nginx --image=nginx --restart=Never --dry-run > 4.yaml
```

5.提供一个pod的yaml，要求添加Init Container，Init Container的作用是创建一个空文件，pod的Containers判断文件是否存在，不存在则退出
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: apline
    image: nginx
    command: ['sh', '-c', 'if [ ! -e "/opt/myfile" ];then exit; fi;']
  ###增加init Container####
  initContainers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'touch /opt/myfile;']
```

6.指定在命名空间内创建一个pod名称为test，内含四个指定的镜像nginx、redis、memcached、busybox

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
  namespace: linjb
spec:
  containers:
  - image: nginx
    name: nginx 
  - image: redis
    name: redis
  - image: busybox
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

7.创建一个pod名称为test，镜像为nginx，Volume名称cache-volume为挂在在/data目录下，且Volume是non-Persistent的

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
spec:
  containers:
  - image: nginx
    name: test
    resources: {}
    volumeMounts:
    - mountPath: /data
      name: cache-volume
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: cache-volume
    emptyDir: {}
status: {}
```

8.列出Service名为test下的pod 并找出使用CPU使用率最高的一个，将pod名称写入文件中



