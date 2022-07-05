# Dependency requirements:
| Plugin | README |
| ------ | ------ |
| kubectl | [https://kubernetes.io/docs/reference/kubectl/] |
| kubens | [https://github.com/ahmetb/kubectx] | 

## Setup namespace for elastic stack
``` 
> kubectl create ns elastic
> kubens elastic << This will change tje name space to elastic and all the command execurte after this will run under elastic namespace
```

## 1. ELATICSEARCH

### Create a secrets file for elasticsearch login
- Create file for username and password
```
> echo -n 'admin' > ./username.txt
> echo -n '1f2d1e2e67df' > ./password.txt
```
- Generate k8s secrets using files
```
kubectl create secret generic db-user-pass \
  --from-file=./username.txt \
  --from-file=./password.txt
```

### Define number of replicas for ES
By default project will setup 3 repolicas for elasticseach called elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2. if you want to make change in that you can do this as below 

- change replicas count from 3 to any number you desire
- You also needs to change cluster.initial_master_nodes env value accourding to the replica count you provided
