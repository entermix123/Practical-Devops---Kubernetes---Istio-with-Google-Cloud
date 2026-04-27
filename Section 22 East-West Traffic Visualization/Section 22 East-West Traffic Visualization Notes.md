# Section 22 East-West Traffic Visualization

## Content
- 85 [Kiali](#85-kiali)
- 86 [Using Kiali](#86-using-kiali)
- 87 [Tips: Securing Kiali](#87-tips-securing-kiali)

We use the created minikube cluster from section 20 Istio Service Mesh for East-West Traffic with installed kube-prometheus stack, HAProxy Ingress Controller, Istio service mesh, cert manager, opentelemtry and Jaeger.

Start the cluster

    bash --> minikube start

Start minikube tunnel and don't close the terminal

    bash --> minikube tunnel

Make sure that address are added to Windows host list

- Open PowerShell as Admin

        PowerShell --> notepad C:\Windows\System32\drivers\etc\hosts

- add

```text
127.0.0.1 localhost                     # required
127.0.0.1 jaeger.local                  # required
127.0.0.1 kiali                         # required
127.0.0.1 blue.devops.local
127.0.0.1 yellow.devops.local           
127.0.0.1 api.devops.local
127.0.0.1 monitoring.devops.local
127.0.0.1 rabbitmq.devops.local
127.0.0.1 chartmuseum.local
127.0.0.1 harbor.local 
127.0.0.1 argocd.local
```

- save the file and exit

## 85 Kiali

[⬆ Back to top](#top)

Kiali is a visualization tool for Istio. It gathers Istio data and arranges it into a graph, so we can know the connection among nodes, including traffic flow, which nodes are unhealthy, etc. We can also configure Istio using Kiali, including disconnecting nodes from traffic. For this reason, we should put Kiali behind a set of credentials. Currently, Kiali uses a service account token to secure itself. For the course, I will turn off security, just for simplicity. 

Please note that Kiali took data from Prometheus. Hence, we must already be scraping Prometheus data using pod and service monitors. Otherwise, Kiali cannot visualize the east-west traffic.

Open file values-kiali-server.yml in the folder istio-05-kiali. By default, Kiali uses a service account token to secure itself. But for simplicity, we will disable this one and use anonymous. Let's expose Kiali via an Ingress controller. The external services section is for connecting Kiali to other tools. We need to set the URL for Prometheus, Jaeger, and Grafana. You need to adjust the external URL address to use your own ingress controller IP address. 

values-kiali-server.yml

```yaml
auth:
  strategy: anonymous           # disabled token credentials

deployment:
  ingress:
    enabled: true
    class_name: haproxy
  resources:
    limits:
      cpu: "0.3"
      memory: 512Mi

external_services:
  prometheus:
    url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus
    internal_url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus
    external_url: http://127.0.0.1/prometheus
    health_check_url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus/-/healthy
  tracing:
    enabled: false
    internal_url: http://jaeger-simple-query.istio-system:16686/jaeger
    external_url: http://127.0.0.1/jaeger
    use_grpc: false
  grafana:
    enabled: true
    internal_url: http://kube-prometheus-stack-grafana.istio-system/grafana
    external_url: http://127.0.0.1/grafana
    health_check_url: http://kube-prometheus-stack-grafana.istio-system/grafana/api/health
    auth:
      type: basic
      username: admin
      password: changeme
```

Install Kiali using Helm. As usual, see the script from the last lecture of the course on resources and references. 

Add and update Kiali helm repo locally

    CMD --> helm repo add kiali https://kiali.org/helm-charts

    # result: "kiali" has been added to your repositories

    CMD --> helm repo update

    # result: Update Complete. ⎈Happy Helming!⎈

Install Kiali server v2.20.0

    CMD --> helm upgrade --install kiali-server kiali/kiali-server  --namespace istio-system --create-namespace --values values-kiali-server.yml --version 2.20.0

    # resutly:
    Release "kiali-server" does not exist. Installing it now.
    NAME: kiali-server
    LAST DEPLOYED: Sat Mar 28 17:52:01 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    Welcome to Kiali! For more details on Kiali, see: https://kiali.io
    The Kiali Server [v2.20.0] has been installed in namespace [istio-system]. It will be ready soon.
    ===============

    (Helm: Chart=[kiali-server], Release=[kiali-server], Version=[2.20.0])

Wait a while and see if it runs. Try accessing it at the ingress address - http://localhost/kiali.

<img src="pics/kiali-ui.png" width="1400" />
<br>
<br>

Then, we need to populate metrics to be sent to Prometheus. Open the file 'istio-prometheus-telemetry.yml'. In this file, we add Prometheus telemetry metrics.

istio-prometheus-telemetry.yml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio
  namespace: istio-system
data:
  mesh: |-
    enableTracing: true
    defaultConfig:
      tracing: {}
    extensionProviders:
    - name: jaeger
      opentelemetry:
        port: 4317
        service: jaeger.jaeger.svc.cluster.local

---

apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 100.0
  metrics:
  - providers:
    - name: prometheus    
```

Apply this file.

    CMD --> kubectl apply -f istio-prometheus-telemetry.yml

    # result:
    configmap/istio unchanged
    telemetry.telemetry.istio.io/mesh-default configured

[⬆ Back to top](#top)


## 86 Using Kiali

[⬆ Back to top](#top)

Before we start, I would like to inform you that the application's user interface may change from time to time. The one you saw in this video might have a different layout or features compared to the latest release. Don't worry, the course should still be relevant, unless there is a major change in the application. In such a case, I will do my best to update the course.

First, let's deploy the sample application.

Delete the devops namepsace

    CMD --> kubectl delete namespace devops

    # result: namespace "devops" deleted

Deploy the sample application

    CMD --> kubectl apply -f devops-istio-basic-deployment-2.0.1.yml

    # result:
    namespace/devops created
    deployment.apps/istio-basic-deployment-blue created
    deployment.apps/istio-basic-deployment-yellow created
    deployment.apps/istio-basic-deployment-white created
    service/devops-blue-clusterip created
    service/devops-yellow-clusterip created
    service/devops-white-clusterip created
    ingress.networking.k8s.io/ingress-istio-basic-haproxy-blue created
    servicemonitor.monitoring.coreos.com/devops-blue-servicemonitor created
    servicemonitor.monitoring.coreos.com/devops-yellow-servicemonitor created
    servicemonitor.monitoring.coreos.com/devops-white-servicemonitor created

Make sure minikube tunnel is up:

    CMD --> minikube tunnel

Let's go back to Postman and run the chain collection. Make sure that for this demo, all response codes on the parameter are 2xx. So we got good traffic.

Chan call 1:

<img src="pics/postman-kiali-chain-1.png" width="1400" />
<br>
<br>

Chan call 2:

<img src="pics/postman-kiali-chain-2.png" width="1400" />
<br>
<br>

Chan call 3:

<img src="pics/postman-kiali-chain-3.png" width="1400" />
<br>
<br>

I will skip the delay endpoint and run each collection with a 0.5-second delay in Postman.

<img src="pics/postman-traffic.png" width="1400" />
<br>
<br>

Open the Kiali user interface - http://localhost/kiali. In the overview, we will see the DevOps namespace, where our applications and east-west traffic reside.

<img src="pics/kiali-devops-namespace-overview.png" width="1400" />
<br>
<br>

Since our application is in the DevOps namespace, we will open it and see that there are 3 applications: blue, white, and yellow. At this point, all should be healthy.

<img src="pics/kiali-devops-namespace-overview-2.png" width="1200" />
<br>
<br>

Then see the graph menu. If you are not seeing anything, try adjusting the time range and make sure you select the DevOps namespace. I will also enable auto refresh every 15 seconds. That is the Prometheus pod monitoring interval, so we will get updated data roughly at the same time as the pod scrapes Prometheus metrics. There are several graph types. At this time, the versioned app graph is the default. So this graph shows how traffic flows among nodes. 

<img src="pics/kiali-devops-namespace-overview-3.png" width="1400" />
<br>
<br>

In other words, it shows a connection among nodes. But we have a lot of nodes, even though we only have 3 apps: blue, yellow, and white. See the legend here. These triangles represent services. 

<img src="pics/kiali-devops-namespace-overview-4.png" width="1400" />
<br>
<br>

The application is located behind the service, so the graph naturally shows it this way. And in the navigation menu, we still have many buttons. For example, we can see various layouts. Kiali generates this layout and shows the same data in a different arrangement. In the display drop-down, we can also choose which data we want to display. For example, to view requests per second, we can enable the traffic rate. Each line will then display the request-per-second data. 

<img src="pics/kiali-devops-namespace-overview-5.png" width="1400" />
<br>
<br>

Also, if I want to see only the app, but hide the service node, I can uncheck this service option. 

<img src="pics/kiali-devops-namespace-overview-6.png" width="1400" />
<br>
<br>

Kiali also provides a nice animation that shows traffic. Just beware that using animation may require more CPU and memory. But this is nice, from here I can see that traffic flows from blue to yellow to white. Also, some traffic flows from blue to white. Since we use open telemetry that sends Jaeger data via gRPC, we see that all these nodes send data to Jaeger. 

<img src="pics/kiali-devops-namespace-overview-7.png" width="1400" />
<br>
<br>

If we only want to see HTTP traffic, we can select only HTTP from this traffic drop-down. If we have many microservices and need to know which microservice calls which others, we can use this graph.

<img src="pics/kiali-devops-namespace-overview-8.png" width="1400" />
<br>
<br>

In this section, we can see a graph for a certain time window and set automatic refresh to get the latest data for that window. This Kiali visual shows traffic that keeps flowing. So if we want to examine traffic over a certain period, change this menu.

<img src="pics/kiali-devops-namespace-overview-9.png" width="1400" />
<br>
<br>

And this graph is interactive. For example, if we want to see which traffic involves the yellow node, we can click it, and it will be highlighted while the others remain transparent. This feature is really helpful, since we now see inbound and outbound traffic involving the yellow node. In real life, the service will be more than this course. To improve visibility into the yellow app, double-click the yellow node, and Kiali will focus on inbound and outbound traffic for that node.

<img src="pics/kiali-devops-namespace-overview-10.png" width="600" />
<br>
<br>

In this sample, all connections are green, indicating that the response status codes along those flows are 2xx, or good response codes.

Let's try something bad. Stop the running postman collection.

<img src="pics/postmen-stop-run.png" width="1400" />
<br>
<br>

Now on chain call five, let's change the dummy response code to a random one. 

<img src="pics/postman-random.png" width="1400" />
<br>
<br>

Run the collection again for all chain endpoints, giving a half-second Postman delay. 

<img src="pics/postman-traffic.png" width="1400" />
<br>
<br>

Return on Kiali and wait a while if needed. Then we can see red or yellow lines. Just by looking at the graph, we know that the transitions from blue to yellow and from yellow to white are problematic. But calls from blue to white are fine. We can know the percentage of unsuccessful response codes (4xx or 5xx).

<img src="pics/kiali-devops-namespace-overview-11.png" width="1000" />
<br>
<br>

In the application menu, we can see the deployed applications. We can see the details of one of the applications. 

<img src="pics/kiali-devops-namespace-overview-12.png" width="1200" />
<br>
<br>

We can use the overview tab to see which calls are successful and which are problematic. 

<img src="pics/kiali-devops-namespace-overview-13.png" width="1400" />
<br>
<br>

Also, the percentage of successful or failed traffic, either inbound or outbound. We can see even more detail on the inbound and outbound metrics tab. 

<img src="pics/kiali-devops-namespace-overview-14.png" width="1400" />
<br>
<br>

More graphs showing volume, duration, etc., with zoomable views for each.

<img src="pics/kiali-devops-namespace-overview-15.png" width="1400" />
<br>
<br>

In the traffic tab, we can see a summary of the traffic flowing throughthis particular application. On the left panel, we still have some menu items left. The workload is more complete than the application details. 

<img src="pics/kiali-devops-namespace-overview-16.png" width="1200" />
<br>
<br>

In the overview, we can see a list of application pods and the following traffic graph. 

<img src="pics/kiali-devops-namespace-overview-17.png" width="1200" />
<br>
<br>

<img src="pics/kiali-devops-namespace-overview-18.png" width="1400" />
<br>
<br>

The traffic tab shows the summary of overall traffic. 

<img src="pics/kiali-devops-namespace-overview-19.png" width="1200" />
<br>
<br>

We can see the inbound and outbound graphs for various metrics.

<img src="pics/kiali-devops-namespace-overview-20.png" width="1400" />
<br>
<br>

<img src="pics/kiali-devops-namespace-overview-21.png" width="1400" />
<br>
<br>

We can also see the application log from the containers in the pod. 

<img src="pics/kiali-devops-namespace-overview-22.png" width="1400" />
<br>
<br>

We also have the envoy tab, which has additional subtabs below for envoy-specific features. 

<img src="pics/kiali-devops-namespace-overview-23.png" width="1400" />
<br>
<br>

Let's see the services menu. It has the same metrics as the application and the workload, so we can skip this.

<img src="pics/kiali-devops-namespace-overview-24.png" width="1200" />
<br>
<br>

<img src="pics/kiali-devops-namespace-overview-25.png" width="1400" />
<br>
<br>

The Istio config is still empty; we will see some samples later.

<img src="pics/kiali-devops-namespace-overview-26.png" width="1200" />
<br>
<br>

The mesh tab visualizes the Kiali topology. It shows Kiali, the Connected services, and workloads.

<img src="pics/kiali-devops-namespace-overview-27.png" width="1400" />
<br>
<br>

[⬆ Back to top](#top)


## 87 Tips: Securing Kiali

[⬆ Back to top](#top)

Kiali has more power than just the metrics viewer, as we will see soon For this reason, it is recommended to secure Kiali. The simplest security is using a service account token. By default, Kiali will use the service account named "kiali". So we can create a service account token for it. 

See pod details in Lens

<img src="pics/lens-kiali-service-acc.png" width="1400" />
<br>
<br>

Open folder 'istio securing kiali'. In here, there is a YAML file that defines a secret for the Kiali service account. We put it in the istio-system namespace, and we need to add this annotation to indicate the service account owner. The type for this secret must also be a service-account-token.

kiali-service-account-token.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kiali-secret
  namespace: istio-system
  annotations:
    kubernetes.io/service-account.name: kiali
type: kubernetes.io/service-account-token
```

Apply this file.

    CMD --> kubectl apply -f kiali-service-account-token.yml

    # result: secret/kiali-secret created

Currently, we don't use any authentication in Kiali. We can use the token "strategy" by changing the auth strategy in the values-kiali-server.yml file.

values-kiali-server-secure.yml

```yaml
auth:
  strategy: token

deployment:
  ingress:
    enabled: true
    class_name: haproxy
  resources:
    limits:
      cpu: "0.3"
      memory: 512Mi

external_services:
  prometheus:
    url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus
    internal_url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus
    external_url: http://127.0.0.1/prometheus
    health_check_url: http://kube-prometheus-stack-prometheus.istio-system:9090/prometheus/-/healthy
  tracing:
    enabled: false
    internal_url: http://jaeger-simple-query.istio-system:16686/jaeger
    external_url: http://127.0.0.1/jaeger
    use_grpc: false
  grafana:
    enabled: true
    internal_url: http://kube-prometheus-stack-grafana.istio-system/grafana
    external_url: http://127.0.0.1/grafana
    health_check_url: http://kube-prometheus-stack-grafana.istio-system/grafana/api/health
    auth:
      type: basic
      username: admin
      password: changeme
```

Then update the Helm installation. 

    CMD --> helm upgrade --install kiali-server kiali-server --repo https://kiali.org/helm-charts  --namespace istio-system --create-namespace --values values-kiali-server-secure.yml

    # result:
    Release "kiali-server" has been upgraded. Happy Helming!
    NAME: kiali-server
    LAST DEPLOYED: Sat Mar 28 20:41:38 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 2
    DESCRIPTION: Upgrade complete
    TEST SUITE: None
    NOTES:
    Welcome to Kiali! For more details on Kiali, see: https://kiali.io
    The Kiali Server [v2.23.0] has been installed in namespace [istio-system]. It will be ready soon.
    ===============

    (Helm: Chart=[kiali-server], Release=[kiali-server], Version=[2.23.0])

Wait a while access the Kiali server again, and now we need to enter the service account token - http://localhost/kiali/console.

<img src="pics/kiali-token-login.png" width="1000" />
<br>
<br>

It is easy to copy the service account token using Lens. Open the kiali-secret in the istio-system namespace, and show the token. Note that we must click this button to display the plain token rather than its base64 encoding. 

<img src="pics/kiali-token.png" width="1400" />
<br>
<br>

We can use this token to log in - http://localhost/kiali/console.

<img src="pics/kiali-token-login-result.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)
