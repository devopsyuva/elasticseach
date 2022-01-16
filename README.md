# Deploy ElasticSearch HighAvalibility on Kubernetes
- Ensure we are deploying Elasticsearch with 3 replicas as HA using Statefulset Object on K8S.
- Headless service to access es.
- As of now, we have used PV as NFS server with accessmode as "ReadWriteOnce".
- PVC will be created dynamically by Statefulset.
- Simple commands to validate:
```
# Create a temparory POD using public Image dubreddy/utils:latest to run curl commands against elasticsearch service on port 9200
- kubectl run -ti --rm --image dubareddy/utils:latest -- bash
- curl http://elasticsearch:9200/_cluster/health?pretty
- curl http://elasticsearch:200/_cluster/state?pretty
- exit

# To ensure, HA works. Lets bring down sts pods to 1 and run above commands to check the health and state of the elasticsearch cluster
- kubectl scale --replicas=1 sts/elasticsearch

# Once testing done, bring back the count to 3 as expected
- kubectl scale --replicas=3 sts/elasticsearch
```
- Sample output while creating Elasticsearch objects(PV, Statefulset and Service) on kubernetes
```
root@controlplanenode:~/elasticsearch# kubectl create -f es-pv.yaml 
persistentvolume/efk-data-volume0 created
persistentvolume/efk-data-volume1 created
persistentvolume/efk-data-volume2 created
root@controlplanenode:~/elasticsearch# kubectl get pv,pvc
NAME                                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
persistentvolume/efk-data-volume0   10Gi       RWO            Retain           Available           nfs-class               4s
persistentvolume/efk-data-volume1   10Gi       RWO            Retain           Available           nfs-class               4s
persistentvolume/efk-data-volume2   10Gi       RWO            Retain           Available           nfs-class               4s
root@controlplanenode:~/elasticsearch#

# Lets create a statefulset to user PV created above:
root@controlplanenode:~/elasticsearch# kubectl create -f es-statefulset.yaml 
statefulset.apps/elasticsearch created
root@controlplanenode:~/elasticsearch# kubectl get pv,pvc
NAME                                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
persistentvolume/efk-data-volume0   10Gi       RWO            Retain           Bound    default/data-elasticsearch-0   nfs-class               5m44s
persistentvolume/efk-data-volume1   10Gi       RWO            Retain           Bound    default/data-elasticsearch-1   nfs-class               5m44s
persistentvolume/efk-data-volume2   10Gi       RWO            Retain           Bound    default/data-elasticsearch-2   nfs-class               5m44s

NAME                                         STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/data-elasticsearch-0   Bound    efk-data-volume0   10Gi       RWO            nfs-class      4m16s
persistentvolumeclaim/data-elasticsearch-1   Bound    efk-data-volume1   10Gi       RWO            nfs-class      4m9s
persistentvolumeclaim/data-elasticsearch-2   Bound    efk-data-volume2   10Gi       RWO            nfs-class      4m2s
root@controlplanenode:~/elasticsearch# kubectl get all -o wide
NAME                             READY   STATUS    RESTARTS      AGE     IP                NODE              NOMINATED NODE   READINESS GATES
pod/elasticsearch-0              1/1     Running   0             4m21s   192.168.101.126   computeplaneone   <none>           <none>
pod/elasticsearch-1              1/1     Running   0             4m14s   192.168.101.127   computeplaneone   <none>           <none>
pod/elasticsearch-2              1/1     Running   0             4m7s    192.168.101.65    computeplaneone   <none>           <none>
pod/static-web-computeplaneone   1/1     Running   5 (12m ago)   2d4h    192.168.101.124   computeplaneone   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d4h   <none>

NAME                             READY   AGE     CONTAINERS      IMAGES
statefulset.apps/elasticsearch   3/3     4m21s   elasticsearch   docker.elastic.co/elasticsearch/elasticsearch:7.16.3
root@controlplanenode:~/elasticsearch# 

root@controlplanenode:~/elasticsearch# kubectl create -f es-service.yaml 
service/elasticsearch created
root@controlplanenode:~/elasticsearch# kubectl get all -o wide
NAME                             READY   STATUS    RESTARTS      AGE     IP                NODE              NOMINATED NODE   READINESS GATES
pod/elasticsearch-0              1/1     Running   0             5m30s   192.168.101.126   computeplaneone   <none>           <none>
pod/elasticsearch-1              1/1     Running   0             5m23s   192.168.101.127   computeplaneone   <none>           <none>
pod/elasticsearch-2              1/1     Running   0             5m16s   192.168.101.65    computeplaneone   <none>           <none>
pod/static-web-computeplaneone   1/1     Running   5 (13m ago)   2d4h    192.168.101.124   computeplaneone   <none>           <none>

NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE    SELECTOR
service/elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   7s     app=elasticsearch
service/kubernetes      ClusterIP   10.96.0.1    <none>        443/TCP             4d4h   <none>

NAME                             READY   AGE     CONTAINERS      IMAGES
statefulset.apps/elasticsearch   3/3     5m30s   elasticsearch   docker.elastic.co/elasticsearch/elasticsearch:7.16.3
root@controlplanenode:~/elasticsearch#
```

- As part of log monitoring, lets deloy fluentd and kibana for visualization as mentioned below:
```
- kubectl apply -f fluentdv1.14.3/
- kubectl apply -f kibanav7.16.3/
```
- Sample output of status when all EFK components run:
```
root@controlplanenode:~/elasticsearch# kubectl get all -o wide
NAME                             READY   STATUS    RESTARTS      AGE   IP                NODE               NOMINATED NODE   READINESS GATES
pod/elasticsearch-0              1/1     Running   0             30m   192.168.101.66    computeplaneone    <none>           <none>
pod/elasticsearch-1              1/1     Running   0             30m   192.168.101.68    computeplaneone    <none>           <none>
pod/elasticsearch-2              1/1     Running   0             30m   192.168.101.71    computeplaneone    <none>           <none>
pod/fluentd-9h879                1/1     Running   0             27m   192.168.121.106   controlplanenode   <none>           <none>
pod/fluentd-rq9wj                1/1     Running   0             27m   192.168.101.65    computeplaneone    <none>           <none>
pod/kibana-7544cb96bf-45r7j      1/1     Running   0             22m   192.168.101.70    computeplaneone    <none>           <none>
pod/static-web-computeplaneone   1/1     Running   8 (40m ago)   4d    192.168.101.97    computeplaneone    <none>           <none>

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE   SELECTOR
service/elasticsearch   ClusterIP   None          <none>        9200/TCP,9300/TCP   30m   app=elasticsearch
service/kibana          NodePort    10.102.24.8   <none>        5601:31784/TCP      22m   app=kibana
service/kubernetes      ClusterIP   10.96.0.1     <none>        443/TCP             6d    <none>

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS   IMAGES                                                                        SELECTOR
daemonset.apps/fluentd   2         2         2       2            2           <none>          27m   fluentd      fluent/fluentd-kubernetes-daemonset:v1.14.3-debian-elasticsearch7-amd64-1.0   app=fluentd

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                   SELECTOR
deployment.apps/kibana   1/1     1            1           22m   kibana       docker.elastic.co/kibana/kibana:7.16.3   app=kibana

NAME                                DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                                   SELECTOR
replicaset.apps/kibana-7544cb96bf   1         1         1       22m   kibana       docker.elastic.co/kibana/kibana:7.16.3   app=kibana,pod-template-hash=7544cb96bf

NAME                             READY   AGE   CONTAINERS      IMAGES
statefulset.apps/elasticsearch   3/3     30m   elasticsearch   docker.elastic.co/elasticsearch/elasticsearch:7.16.3
root@controlplanenode:~/elasticsearch#
```
