# Dependency requirements:
| Plugin | README |
| ------ | ------ |
| kubectl | https://kubernetes.io/docs/reference/kubectl/ |
| kubens | https://github.com/ahmetb/kubectx | 

## Setup namespace for elastic stack
``` 
❯ kubectl create ns elastic
❯ kubens elastic << This will change the name space to elastic and all the command execurte after this will run under elastic namespace
```

## 1. ELATICSEARCH

### Create a secrets file for elasticsearch login
- Create file for username and password
```
❯ echo -n 'elastic' > ./username
❯ echo -n 'password' > ./password
```
- Generate k8s secrets using files
```
❯ kubectl create secret generic elasticsearch-master-credentials \
  --from-file=./username \
  --from-file=./password
```
```
❯ k describe secrets/elasticsearch-master-credentials
Name:         elasticsearch-master-credentials
Namespace:    elastic
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
username:  8 bytes

```

if you want to decode your password later on you can do this by `kubectl get secrets/elasticsearch-master-credentials --template={{.data.password}} | base64 -D` 

### Define number of replicas for ES
By default project will setup 3 repolicas for elasticseach called elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2. if you want to make change in that you can do this as below 

- change replicas count from 3 to any number you desire
- You also needs to change cluster.initial_master_nodes env value accourding to the replica count you provided

### Apply and validate ES
```
❯ kubectl apply -f elasticsearch
poddisruptionbudget.policy/elasticsearch-master-pdb created
secret/elasticsearch-master-certs created
service/elasticsearch-master created
service/elasticsearch-master-headless created
statefulset.apps/elasticsearch-master created
```

```
❯ kubectl get pods  -l app=elasticsearch-master
NAME                     READY   STATUS    RESTARTS   AGE
elasticsearch-master-0   1/1     Running   0          1m
elasticsearch-master-1   1/1     Running   0          1m
elasticsearch-master-2   1/1     Running   0          2m

```
- Port-Forward service using `kubectl port-forward service/elasticsearch-master 9200`
- Ppen browser and access `https://localhost:9200` ( if will show certificate error but you can ignore it !)


## 2. KIBANA
### Generate Service Account token
- Port-Forward elasticsearch using `kubectl port-forward service/elasticsearch-master 9200`
- Send CURL requect to genrate service account token
```
❯ curl -u elastic:<password> -kX POST "https://localhost:9200/_security/service/elastic/kibana/credential/token?pretty"
{
  "created" : true,
  "token" : {
    "name" : "token_usXEzYEBpYRDHGxccfI_",
    "value" : "AAEAAWVsYXN0aWMva2liYW5hL3Rva2VuX3VzWEV6WUVCcFlSREhHeGNjZklfOlVWNDFtN0RuU2lLT25SMTVIRzRKNkE"
  }
}
```
- store the value of response in serviceAccountToken filename
- generate any random encription key and store in encryptionkey filename
- Create kibana secrets
```
kubectl create secret generic kibana \
  --from-file=./encryptionkey \
  --from-file=./serviceAccountToken
```
- Apply and validate Kibana
```
❯ k apply -f kibana
configmap/kibana-config created
deployment.apps/kibana created
service/kibana created
```

```
❯ kubectl get pods  -l app=kibana
NAME                      READY   STATUS    RESTARTS   AGE
kibana-84fbd79c4c-vmngw   1/1     Running   0          12m
```
- if all pods are in Running state then we can port-forward kibana using `kubectl port-forward service/kibana 5601` and open url http://localhost:5601

## 3. LOGSTASH
- Apply and validate Logstash
```
❯ kubectl apply -f logstash
configmap/logstash-config created
configmap/logstash-pipeline created
poddisruptionbudget.policy/logstash-pdb created
service/logstash-headless created
statefulset.apps/logstash created
```

```
❯ kubectl get pods  -l app=logstash
NAME                      READY   STATUS    RESTARTS   AGE
kibana-84fbd79c4c-vmngw   1/1     Running   0          12m
```
## 4. FILEBEAT
- Apply and validate FileBeat
```
❯ kubectl apply -f filebeat
configmap/filebeat-daemonset-config created
daemonset.apps/filebeat created
clusterrole.rbac.authorization.k8s.io/filebeat-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/filebeat-cluster-role-binding created
role.rbac.authorization.k8s.io/filebeat-role created
rolebinding.rbac.authorization.k8s.io/filebeat-role-binding created
serviceaccount/filebeat created
```

```
❯ kubectl get pods  -l app=filebeat
```
## 5. METRICBEAT
- Apply and validate metricbeat
```
❯ kubectl apply -f metricbeat
configmap/metricbeat-daemonset-config created
configmap/metricbeat-deployment-config created
daemonset.apps/metricbeat created
deployment.apps/metricbeat created
serviceaccount/metricbeat-kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/metricbeat-kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/metricbeat-kube-state-metrics created
service/metricbeat-kube-state-metrics created
deployment.apps/metricbeat-kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/metricbeat-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/metricbeat-cluster-role-binding created
role.rbac.authorization.k8s.io/metricbeat-role created
rolebinding.rbac.authorization.k8s.io/metricbeat-role-binding created
serviceaccount/metricbeat created
```
- Create Dahboards of metricbeat
```
❯ kubectl exec -it pod/metricbeat-metrics-779489d9df-zj7rb -- bash

# metricbeat setup --dashboards \
  -E setup.kibana.host=kibana:5601 \
  -E setup.kibana.username=elastic \
  -E setup.kibana.password=<password>
```
