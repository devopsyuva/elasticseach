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
