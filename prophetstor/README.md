
Kubernetes HPA Autoscaling with Kafka metrics
=============================================

# Apply deployment

``$ oc apply -f .``

# Load k8s-prometheus-adapter image or use Prophetstor images repository

``docker load -i k8s-prometheus-adapter-amd64.tgz``  
``docker tag directxman12/k8s-prometheus-adapter-amd64:latest  docker-registry.default.svc:5000/default/k8s-prometheus-adapter-amd64:latest``  
``docker login -u openshift -p $(oc whoami -t) docker-registry.default.svc:5000``  
``docker push docker-registry.default.svc:5000/default/k8s-prometheus-adapter-amd64:latest``  

# Edit k8s-prometheus-adapter deployment file image path

``$ oc edit deployemnt prometheus-adapter -n default``

 ## Correct image path
 image: docker-registry.default.svc:5000/default/k8s-prometheus-adapter-amd64:latest    
 or  
 image: repo.prophetservice.com/k8s-prometheus-adapter-amd64:latest

## Assign service account
``$ oc adm policy add-cluster-role-to-user admin system:serviceaccount:default:custom-metrics-apiserver`` 

## Verify the custom api
### Verify the custom api provide default resource 
``$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/``
### Verify the custom api provde the kefka lag metrics
``$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/*/metrics/kafka_consumergroup_lag``

## Config the kafka hpa with the kafka lag metrics
``$ oc apply -f ./customer-hpa-kafka.yaml ``

### constomer-hpa-kafka.yaml
```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v1
metadata:
  name: consumer-hpa
  namespace: myproject
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: consumer
    namespace: myproject
  minReplicas: 1
  maxReplicas: 40
  metrics:
  - type: Pods
    pods:
      metricName: "kafka_consumergroup_lag"
      targetAverageValue: 1000
 ```

### Use commend to get hpa status and verify the reference target value that should associate lag metrics
``$ oc get hpa consumer-hap``
