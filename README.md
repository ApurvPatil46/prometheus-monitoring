# prometheus-monitoring

## Monitoring cluster with Prometheus and node_exporter

### Workflow:- 
  1. Setup alertmanager
  2. Setup node_exporter
  3. Setup prometheus
  4. Create alert.rules to monitor cluster
  5. Configure alertmanager , node_exporter and alert.rules with prometheus for end-to-end monitoring service deployment

## Approach
  1. I have created an environment where all the deployment , services and configmaps supposed to reside.
  2. Then I have proceed with creating persistent volumes and persistent volume claims to keep track of or day and also it's persistence. The more information about persistent volumes and persistent volume claims can be found here  `https://kubernetes.io/docs/concepts/storage/persistent-volumes/`.
  3. Then I came up with prometheus service. Here we are using **NodePort** service in order to access service from outside world. (Bydefault service is ClusterIP)
  4. After that I have created prometheus configuration file. As name suggest all the configuration are resides inside file regarding to targets to monitor.
  5. In the last we have deployed prometheus deployment.

### Steps

  1. **Monitoring Namespace** - In this yaml file we are deploying environment that will accumulate all the deployments regarding to prometheus monitoring.

      In order to deploy namespace use command - 

      `$ kubectl apply -f monitoring-namespace.yaml`
  2. **Persistent Volumes and Persistent Volume Claims** - In this yaml file we are providing **STORAGE** to our prometheus deployment in order to store data and persist it.
      In order to deploy persistent volumes and prsistent volume claims use command - 
      
      `$ kubectl apply -f prometheus-pv.yaml`
      `$ kubectl apply -f prometheus-pvc.yaml`
  3. **Prometheus Service** - In this file we are using NodePort service in order to access it from outside world. We are exposing service to internet through port **30000** and service is listening to prometheus' **9090** port.
      In oerder to deploy prometheus service use command - 
      
      `$ kubectl apply -f prometheus-service.yaml`
  4. **Prometheus Config File** - In this file we have provided targets to prometheus in order to monitor and listen continuously to them.
      In order to deploy configuration file use command - 
      
      `$ kubectl apply -f prometheus-config.yaml`
  5. **Prometheus Deployment** - With this we are deploying prometheus pod that will configure with services all the ports that are given and take all the files created above to run in an environment.
      In order to deploy prometheus pods and deployments use command -
      
      `$ kubectl apply -f prometheus-deployment.yaml`
      
### Note
```
    Since are using persistent volumes and claims they are supposed to deployed before actual prometheus deployment 
    so that when you deploy prometheus it will get configure with PV & PVCs. Else it will throw error `CrashLoopBackOff`.
```   
### Future enhancement

  1. Node_exporter service and alarms are not yet configured with prometheus configuration file. That supposed to get done. 
  2. Alert.rules yet to create in order to monitor **cluster health** . 
  3. Later I'm looking after deploying these services on `AWS EKS` to check if they are working as expected.
  4. In the last step I'll deploy multiple clusters and try to monitor them through single in order to acheive centralized monitoring as end goal.

