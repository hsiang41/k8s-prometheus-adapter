
Kubernetes HPA Autoscaling with Kafka metrics
=============================================

# Apply deployment

``$ oc apply -f .``

# Edit k8s-prometheus-adapter deployment file image path

``$ oc edit deployemnt prometheus-adapter -n default``

 ## Correct image path
 image: repo.prophetservice.com/k8s-prometheus-adapter-amd64:latest

## Assign service account
``$ oc adm policy add-cluster-role-to-user admin system:serviceaccount:default:custom-metrics-apiserver`` 

## Verify the custom api
### Verify the custom api provide default resource 
``$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/``
```console
[root@node17121 ~]# oc get --raw /apis/custom.metrics.k8s.io/v1beta1
{"kind":"APIResourceList","apiVersion":"v1","groupVersion":"custom.metrics.k8s.io/v1beta1","resources":[{"name":"namespaces/kafka_consumergroup_lag","singularName":"","namespaced":false,"kind":"MetricValueList","verbs":["get"]},{"name":"pods/kafka_consumergroup_lag","singularName":"","namespaced":true,"kind":"MetricValueList","verbs":["get"]}
```

### Verify the custom api provide the kafka lag metrics
``$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/myproject/pods/*/kafka_consumergroup_lag``
```console
[root@node17121 ~]# oc get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/myproject/pods/*/kafka_consumergroup_lag
{"kind":"MetricValueList","apiVersion":"custom.metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/custom.metrics.k8s.io/v1beta1/namespaces/myproject/pods/%2A/kafka_consumergroup_lag"},"items":[{"describedObject":{"kind":"Pod","namespace":"myproject","name":"my-cluster-kafka-exporter-5995bf9d67-clfcb","apiVersion":"/v1"},"metricName":"kafka_consumergroup_lag","timestamp":"2020-03-16T06:03:40Z","value":"2335363m","selector":null}]}
```
## Config the kafka hpa with the kafka lag metrics
``$ oc apply -f ./customer-hpa-kafka.yaml ``

### customer-hpa-kafka.yaml
```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: consumer-hpa
  namespace: myproject
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: consumer
    namespace: myproject
  minReplicas: 1
  maxReplicas: 40
  metrics:
  - type: Pods
    pods:
      metricName: "kafka_consumergroup_lag"
      targetAverageValue: 10000
 ```

### Use command to get hpa status and verify the reference target value that should match lag metrics
``$ oc get hpa consumer-hpa``
```console
[root@node17121 config]# oc get hpa
NAME           REFERENCE             TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
consumer-hpa   Deployment/consumer   1000/1300   1         40        5          2m
```

# Reference
https://github.com/DirectXMan12/k8s-prometheus-adapter