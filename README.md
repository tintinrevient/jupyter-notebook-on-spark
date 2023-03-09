# Jupyter Notebook on Spark

## Prerequisites
- Kubernetes 1.19+
- Helm 3.2.0+

## Installation

1. Create the [pyspark-notebook](https://hub.docker.com/r/jupyter/pyspark-notebook/tags) deployment and its headless service ([why headless service?](https://stackoverflow.com/questions/52707840/what-is-a-headless-service-what-does-it-do-accomplish-and-what-are-some-legiti)):
```bash
kubectl create -f pyspark-notebook.yaml
```

The created resources are as below after checking `kubectl get all -n spark`:
```bash
pod/my-notebook-deployment-5b6bb74fbf-xkfxm   1/1     Running             0          80m

NAME                          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/my-notebook-service   ClusterIP   None         <none>        29413/TCP   80m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-notebook-deployment   1/1     1            1           80m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/my-notebook-deployment-5b6bb74fbf   1         1         1       80m
```

2. Expose the `jupyter notebook` and log into it with the token found out in `kubectl exec -it my-notebook-deployment-5b6bb74fbf-xkfxm -n spark bash` then `jupyter server list`:
```bash
kubectl port-forward -n spark deployment.apps/my-notebook-deployment 8888:8888
```

3. Create the service account `spark`:
```bash
kubectl create serviceaccount spark -n spark
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=spark:spark --namespace=spark
```

4. Create the spark session in `jupyter notebook`:
```bash
from pyspark import SparkContext, SparkConf
from pyspark.sql import SparkSession

# Create Spark config for our Kubernetes based cluster manager.
sparkConf = SparkConf()
sparkConf.setMaster("k8s://https://kubernetes.default.svc.cluster.local:443")
sparkConf.setAppName("spark")
sparkConf.set("spark.kubernetes.container.image", "apache/spark-py:v3.3.2")
sparkConf.set("spark.kubernetes.namespace", "spark")
sparkConf.set("spark.executor.instances", "1")
sparkConf.set("spark.executor.cores", "2")
sparkConf.set("spark.driver.memory", "512m")
sparkConf.set("spark.executor.memory", "512m")
sparkConf.set("spark.kubernetes.pyspark.pythonVersion", "3.9")
sparkConf.set("spark.kubernetes.authenticate.driver.serviceAccountName", "spark")
sparkConf.set("spark.kubernetes.authenticate.serviceAccountName", "spark")
sparkConf.set("spark.driver.port", "29413")
sparkConf.set("spark.driver.host", "my-notebook-service.spark.svc.cluster.local")

# Initialize our Spark cluster, this will actually generate the worker nodes.
spark = SparkSession.builder.config(conf=sparkConf).getOrCreate()
sc = spark.sparkContext
```

5. Destroy the above environment by `kubectl delete namespace spark`.

## References
* https://spark.apache.org/docs/latest/running-on-kubernetes.html
* https://spark.apache.org/docs/latest/spark-standalone.html
* https://spark.apache.org/downloads.html
* https://hub.docker.com/r/apache/spark-py/tags
* https://hub.docker.com/r/jupyter/pyspark-notebook/tags
* https://towardsdatascience.com/ignite-the-spark-68f3f988f642
* https://stackoverflow.com/questions/65980391/spark-executors-fails-to-run-on-kubernetes-cluster