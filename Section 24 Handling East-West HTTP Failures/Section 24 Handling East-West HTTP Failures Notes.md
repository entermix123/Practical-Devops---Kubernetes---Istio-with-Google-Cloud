# Section 24 Handling East-West HTTP Failures

## Content
- 100 [HTTP Timeout](#100-http-timeout)
- 101 [HTTP Retry](#101-http-retry)
- 102 [Circuit Braker - Theory](#102-circuit-braker---theory)
- 103 [Circuit Braker - Hands On](#103-circuit-braker---hands-on)

We use the created minikube cluster from section 20 Istio Service Mesh for East-West Traffic and 21 Distributed Tracing with installed kube-prometheus stack, HAProxy Ingress Controller, Istio service mesh, cert manager, opentelemtry and Jaeger.

Start the cluster

    bash --> minikube start

Start minikube tunnel and don't close the terminal

    bash --> minikube tunnel

Make sure that address are added to Windows host list

- Open PowerShell as Admin

        PowerShell --> notepad C:\Windows\System32\drivers\etc\hosts

- add

```text
127.0.0.1 localhost
127.0.0.1 jaeger.local
127.0.0.1 kiali
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

<img src="pics/name.png" width="800" />
<br>
<br>

## 100 HTTP Timeout

[⬆ Back to top](#top)

For the HTTP timeout case, we will try on yellow. Remove all deployments, pods, services, and ingress from the devops namespace. 

    CMD --> kubectl delete namespace devops

    # result: namespace "devops" deleted

Also, remove the Istio configurations so we can start fresh.

<img src="pics/kiali-no-destination-rules.png" width="1200" />
<br>
<br>

Open the folder "istio-timeout-retry" and apply the file - devops-istio-timeout-retry.yml.

    CMD --> kubectl apply -f devops-istio-timeout-retry.yml

    # result:
    namespace/devops created
    deployment.apps/istio-retry-deployment-blue created
    deployment.apps/istio-retry-deployment-yellow created
    deployment.apps/istio-retry-deployment-white created
    service/devops-blue-clusterip created
    service/devops-yellow-clusterip created
    service/devops-white-clusterip created
    ingress.networking.k8s.io/ingress-istio-retry-nginx-blue created

Wait few minutes so the pods can start

    CMD --> kubectl get pods -n devops

This configuration will create one blue, one yellow, and one white service, along with an ingress. 

Make sure minikube tunnel is up

    CMD --> minikube tunnel

Open Postman and access 'blue-chain-one'. As we can see in the response, a call from blue to yellow will have a random delay in milliseconds. 

<img src="pics/postman-blue-one-result.png" width="1200" />
<br>
<br>

There might be cases where we only wait for a few seconds, and if we don't get a response within that threshold, we need to break the connection. In the other case, the backend was slow and took a long time to complete. If we need to adjust this value, we can do so from Istio.

In Kiali http://localhost/kiali/, open the yellow service and set the HTTP timeout. 

<img src="pics/kiali-yellow-service.png" width="800" />
<br>
<br>

<img src="pics/kiali-yellow-service-time-out.png" width="1200" />
<br>
<br>

For example, we will wait up to 200 milliseconds. If there is no response from thetarget, terminate the request. 

<img src="pics/kiali-yellow-service-time-out-1.png" width="1000" />
<br>
<br>

<img src="pics/kiali-yellow-service-time-out-2.png" width="1000" />
<br>
<br>

<img src="pics/kiali-yellow-service-time-out-3.png" width="1000" />
<br>
<br>

Make sure minikube tunnel is up

    CMD --> minikube tunnel

Now in Postman hit the 'blue-chain-one'. We sometimes do not get a response when the random blue-yellow call delay exceeds 200 milliseconds, because Istio terminates the request after 200 milliseconds. 

<img src="pics/postman-more-than-200ms-delay.png" width="1200" />
<br>
<br>

<img src="pics/postman-less-than-200ms-delay.png" width="1200" />
<br>
<br>

If we change the routing timeout to 3 seconds and call the API again, we will get a response, as yellow already responded before the timeout elapses.

<img src="pics/edit-virtual-service.png" width="1200" />
<br>
<br>

<img src="pics/edit-virtual-service-1.png" width="1200" />
<br>
<br>

<img src="pics/postman-more-than-200ms-delay-2.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)


## 101 HTTP Retry

[⬆ Back to top](#top)

When we learn about the fallacy of distributed systems, we can manage the "network is always reliable" fallacy by using retries. This method means that if the target backend is not responding on the first attempt, retry the call several times. It may be an intermittent problem. In such a case, retrying will likely yield a good response. If after several attempts, the target backend still does not give a good response, then consider the request failed. It may not be a network problem, but a backend problem. To implement this we can use a library in a programming language. For example, Java can use the resilience4j library, Go uses retryable HTTP, and other programming library will have their own library. Although this approach works, it has a drawback. First, all microservices must implement this library. And second, it is language-dependent, which means it requires coding effort to implement or change the retry behaviour. Luckily, Istio can handle this for us, so no coding is needed. And adding or changing is easy. 

We will use the objects from the "istio timeout retry" folder from the previous lesson. Just remove all Istio configurations so we can start fresh.

<img src="pics/remove-istio-configs-1.png" width="1200" />
<br>
<br>

<img src="pics/remove-istio-configs-2.png" width="1200" />
<br>
<br>

From Kiali, open the yellow Service and add request routing. 

<img src="pics/kiali-yellow-service.png" width="800" />
<br>
<br>

<img src="pics/kiali-yellow-service-2.png" width="1000" />
<br>
<br>

Let's add a condition. The condition is that the header X-retry must be present. Then, on the request timeout tab, add HTTP retry. We will try up to 10 times, but nomore than 500 milliseconds per retry. 

<img src="pics/kiali-yellow-service-3.png" width="1000" />
<br>
<br>

<img src="pics/kiali-yellow-service-4.png" width="1000" />
<br>
<br>

Also, add a global route for any request, so requests without a matching condition will still be served. 

<img src="pics/kiali-yellow-service-5.png" width="1000" />
<br>
<br>

<img src="pics/kiali-yellow-service-6.png" width="1000" />
<br>
<br>

Create the configuration.

<img src="pics/kiali-yellow-service-7.png" width="1000" />
<br>
<br>

<img src="pics/kiali-yellow-service-8.png" width="1000" />
<br>
<br>

We can edit the virtual service from Kiali. 

<img src="pics/kiali-yellow-service-9.png" width="800" />
<br>
<br>

In the retryOn parameter, we can specify the HTTP status codes we want to retry. It receives string '5xx' for any 5xx status, but for other statuses, we need to type the exact status code. So, for example, we will add 5xx and various 4xx status codes. Make sure there is no space after a comma. Edit it.

<img src="pics/kiali-yellow-service-10.png" width="800" />
<br>
<br>

Make sure minikube tunnel is up

    CMD --> minikube tunnel

Now open the blue chain four endpoint, and add 5xx as a parameter. We will get an error because this endpoint passes the 5xx request parameter to the yellow call, and the yellow service will return 5xx. 

<img src="pics/kiali-yellow-service-11.png" width="1200" />
<br>
<br>

But how can we be sure that the retry is called? At this point, the retry is not called because we did not pass the X-retry header, which is a prerequisite we defined in Istio.

<img src="pics/kiali-yellow-service-12.png" width="1200" />
<br>
<br>

But to make sure, open Jaeger- http://jaeger.local/. In the distributed tracing, we can see that only one call to blue is executed, and it returns immediately.

<img src="pics/jaeger-trace-yellow-1.png" width="1200" />
<br>
<br>

<img src="pics/jaeger-trace-yellow-2.png" width="1000" />
<br>
<br>

Let's add the X-retry header to the API call. The value can be anything, since we only check whether the header exists. 

<img src="pics/kiali-yellow-service-13.png" width="1200" />
<br>
<br>

We still get a 5xx error, but if we open Jaeger and look at the trace, we can see that it now tries 10 times before giving up.

<img src="pics/jaeger-trace-yellow-3.png" width="1200" />
<br>
<br>

<img src="pics/jaeger-trace-yellow-4.png" width="1200" />
<br>
<br>

Let's try to hit the endpoint with a 'random' parameter. We might need to try a few times to get the result. On a 4xx or 5xx error, the system retries until eventually we get a 2xx or 3xx status, unless after 10 calls, the status is still 4xx or 5xx.

<img src="pics/kiali-yellow-service-14.png" width="1200" />
<br>
<br>

We can check Jaeger and see that the yellow call is retried until it returnsa non-4xx or non-5xx status code.

<img src="pics/jaeger-trace-yellow-5.png" width="1200" />
<br>
<br>

<img src="pics/jaeger-trace-yellow-6.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)


## 102 Circuit Braker - Theory

[⬆ Back to top](#top)

In synchronous communication via an API, a common pattern for handling communication failures is the Circuit Breaker pattern. What is a circuit breaker? It is a small plastic device designed to protect an electrical circuit from damage caused by an electrical fault, such as a short circuit. Its basic function is to interrupt current flow when a fault is detected. A circuit breaker in a microservice has the same function: to interrupt current flow after a fault. The difference is that a communication failure occurred because the target API cannot be accessed. Instead of returning an error message or timing out the client, we can set a circuit breaker that opens when the target service is unavailable for a specified period. After a few moments of a timeout period, the client should try again, and if it succeeds this time, the circuit breaker will be closed. A circuit breaker is a common pattern in microservices, not specific to Istio. You can see the coding implementation explained in my other course, Microservice Architecture and Pattern. You can get a special price for it at the bonus lesson, the last section of this course.

<img src="pics/circuit-braker-explained.png" width="800" />
<br>
<br>

How does a circuit breaker work? Let's say we have a microwave. It has electricity flowing to it. Suddenly, the microwave is broken and could cause electrical damage. 

<img src="pics/circuit-braker-scenario-1.png" width="600" />
<br>
<br>

At this time, the electrical circuit breaker, that small plastic device, will be triggered and cut off the electricity flow. 

<img src="pics/circuit-braker-scenario-2.png" width="600" />
<br>
<br>

The same goes for software circuit breakers. We have traffic from clients or other services. Suppose we have application blue calling application yellow. And we put a circuit breaker at the blue one. This circuit breaker is a common concept, not Istio-specific. Thus, we can use a programming language library to implement it. In the blue application container, we use a programming language library or write our own logic to implement a circuit breaker. For example, if the application is written in Java, we use a Java-specific circuit breaker library, such as Resilience4J. Then we define some indicator for the circuit breaker. For example, it checks based on http status 502, 503, or 504, which indicates a connection problem. 

<img src="pics/circuit-braker-scenario-3.png" width="600" />
<br>
<br>

If such an HTTP status is received 100 times consecutively, within 30 seconds, Then the blue circuit breaker will treat yellow as having a problem and temporarily suspend its connection to Yellow. Note that both blue and yellow still exist. The circuit breaker only temporarily suspends traffic. For example, it will suspend for 2 minutes. 

<img src="pics/circuit-braker-scenario-4.png" width="600" />
<br>
<br>

Then after 2 minutes, The traffic will be reopened. These configurations are part of the application itself, since we use a programming language library embedded in the application code.

<img src="pics/circuit-braker-scenario-5.png" width="600" />
<br>
<br>

Implementing a circuit breaker in the application code has several drawbacks. First, adding, changing, or deleting a circuit breaker will require changes to the application code. Although configuration might be injected from other places, like a Kubernetes environment variable, the implementation itself must be in the application code. If we forget to add the circuit breaker, then the application will not have it. It relies on a programming language library or a custom implementation that might be obsolete. For example, a few years ago, the Java circuit breaker library commonly used by Netflix was Hystrix. But Hystrix became obsolete and is no longer actively developed. If we upgrade the application, it might no longer be compatible with Hystrix. When I record this course, the Resilience4j library is used, which is actively maintained. But who knows how long it will survive, or stay compatible? 

Fortunately, Istio (or Envoy Proxy to be exact) also provides circuit breaker functionality. Implementing a circuit breaker on Envoy means we implement on the infrastructure level, on the sidecar proxy, and does not require any change to the application code. We add, configure, and remove circuit breakers in Istio via YAML configuration, which is easy. It may become obsolete in the future. But when the current Envoy or Istio is obsolete, and we update them, it means we change the whole Istio infrastructure, and circuit breaker implementation is part of the overall update, which, in this case, is more manageable than implementation-dependent changes to tens or hundreds of microservice applications. 

The circuit breaker concept remains the same when we use Istio. We implement a circuit breaker on the blue pod. Remember, we will have an application and Envoy as a sidecar proxy in the pod. We have an indicator and a configurable temporary traffic suspension. The difference is that we don't need to change anything on the application. Instead, we add, configure, or remove a circuit breaker through Istio. In turn, Istio will push that configuration to the Envoy proxy. Since all traffic will go through the sidecar proxy, if we configure properly, the pod will have circuit breaker capability.

<img src="pics/circuit-braker-istio.png" width="600" />
<br>
<br>

A circuit breaker is useful, but using it can be tricky. Let's see a case without a circuit breaker. Suppose blue is calling yellow, and yellow is somehow busy. Yellow is still up, But there is a glitch that causes blue to receive a 504 status, maybe because the process on yellow takes 1 minute to finish, while blue's timeout is 30 seconds. 

<img src="pics/circuit-braker-scenario-6.png" width="600" />
<br>
<br>

The problem is that blue keeps calling yellow, so yellow must process all requests. This case eventually causes the Yellow to overload, and the yellow application crashes. The problem with yellow can be intermittent, perhaps because the database is currently busy or because it's almost reaching its limit. But since requests keep coming in, Yellow is forced to keep serving, which leads to a crash. Of course, when using Kubernetes, we can have multiple pods for high availability on yellow. Even on a single pod, Kubernetes will need some time to restart the yellow automatically. Let's say this "blue keeps calling yellow" is still happening even after Yellow restarts. 

<img src="pics/circuit-braker-scenario-7.png" width="600" />
<br>
<br>

Since each call opens a connection, and some pending processes may be on blue connections, blue connections will stack up, which might consume blue resources, and blue will also go down. 

<img src="pics/circuit-braker-scenario-8.png" width="600" />
<br>
<br>

Applying a circuit breaker means we are applying backpressure to this call. Back pressure is when we terminate the calling flow to avoid target overload (in this case, yellow overload). It's like temporarily closing the pipeline. 

<img src="pics/circuit-braker-scenario-9.png" width="600" />
<br>
<br>

And this is exactly what a circuit breaker does. It blocks traffic to Yellow, so Yellow is not overloaded with calls, giving it some time to recover. 

<img src="pics/circuit-braker-scenario-10.png" width="600" />
<br>
<br>

After some time, the circuit breaker allows the traffic, so blue can call Yellow again. 

<img src="pics/circuit-braker-scenario-9.png" width="600" />
<br>
<br>

Notice that blue will still receive a bad response when the circuit breaker is triggered, because there is no connection to Yellow. It is the caller's responsibility (the blue) to handle this error, even during the traffic block period.

<img src="pics/circuit-braker-scenario-11.png" width="600" />
<br>
<br>

We must be careful when setting the circuit breaker. A circuit breaker that is too delicate might cause frequent, unexpected traffic blocks. For example, if we set the circuit breaker's threshold too low, say 5 consecutive errors in 1 minute. Or the other way around: a circuit breaker with too high a value (like 1,000 consecutive errors in 1 minute) might never be triggered, and back pressure will never be applied. There is no exact formula for this, as traffic and how the service handles requests will vary. Setting circuit breaker value, when needed, is more like specific parameters per traffic pipeline.

[⬆ Back to top](#top)

## 103 Circuit Braker - Hands On

[⬆ Back to top](#top)

A circuit breaker in Istio is called outlier detection, part of the traffic policy in a destination rule. Don't be confused. It is only a term in Istio, but it actually refers to a circuit breaker. However, there are some Istio terms related to outlier detection. The best way to see each term description is in the Istio destination rule documentation, where you can find it in the course's resources and references section. But I will explain some common terms here. 

The one that we must know is ejection. An ejection occurs when the circuit breaker is triggered, and the pipeline is closed, preventing traffic from flowing. There is a term 'base ejection time' that indicates how long traffic should remain blocked when the circuit breaker is triggered. Consecutive gateway errors are the number of consecutive 502, 503, or 504 errors required to trigger the circuit breaker. This error is only specific to response codes 502, 503, and 504. But there are other 5xx errors. The most generic is the 500 status code, which indicates an unknown server error. It happens when an unhandled code exception occurs, such as dividing a number by zero, or the server receives a null value where it should not. If we want to consider other 5xx status codes besides 502, 503 and 504, we can use the parameter consecutive 5xx errors. We can set both consecutive gateway and consecutive 5xx, or only one of them. An interval is the time period for consecutive errors. For example, if we want the circuit breaker indicator to trigger after 100 consecutive errors within a 40-second window, we set the interval to 40. The maximum ejection percentage limits the number of unhealthy pods that can be removed from load balancing simultaneously. It protects the system from ejecting too many pods and causing a complete outage. For example, with 4 pod instances and maxEjectionPercent set to 75%, up to 3 pods can be ejected, so at least 1 pod will still receive traffic. With only 2 pod instances, 75% means only 1 pod can be ejected, so the other pod must remain active. If there is only 1 pod replica, it cannot be ejected because 75% of 1 is less than one full pod, so the system keeps it in rotation to avoid total downtime. This setting ensures some capacity is always available, even when multiple pods fail.

<img src="pics/outlier-detection-basic-settings.png" width="1000" />
<br>
<br>

Clear all deployments, pods, services, and ingress on the devops namespace.

    CMD --> kubectl delete namespace devops

    # result: namespace "devops" deleted

Open the folder istio-circuit-breaker and apply the configuration file - devops-istio-circuit-breaker.yml. This configuration will deploy blue, yellow, and white services. Nothing special.

    CMD --> kubectl apply -f devops-istio-circuit-breaker.yml

    # result:
    namespace/devops created
    deployment.apps/istio-circuit-breaker-deployment-blue created
    deployment.apps/istio-circuit-breaker-deployment-yellow created
    deployment.apps/istio-circuit-breaker-deployment-white created
    service/devops-blue-clusterip created
    service/devops-yellow-clusterip created
    service/devops-white-clusterip created
    ingress.networking.k8s.io/ingress-istio-circuit-breaker-haproxy-blue created

Wait a few minutes for the pods to be running

    CMD --> kubectl get pods -n devops

Make sure minikube tunnel is up

    CMD --> minikube tunnel

In Postman hit the 'chain 4' endpoints several times. Make sure we get a good response. 

<img src="pics/postman-circuit-braker-1.png" width="1200" />
<br>
<br>

Open Kiali - http://localhost/kiali/, and go to the yellow service, where we will add a circuit breaker. Create a request routing to yellow yellow with no rule.

<img src="pics/kiali-yellow-service.png" width="800" />
<br>
<br>

<img src="pics/kiali-yellow-service.png-1.png" width="1200" />
<br>
<br>

Then, in the advanced section, add a circuit breaker. There, we will see two items: a connection pool, which can be used to limit the number of concurrent connections for the yellow service. This feature helps limit the number of connections to the target service. However, we will not use it for now. Add outlier detection. In this version of Kiali, there are very few fields, but that's OK. We will update the YAML later on preview. Add 5 consecutive errors and hit the preview button.

<img src="pics/kiali-yellow-service.png-2.png" width="1200" />
<br>
<br>

Don't forget to change the subset name to any string for both the destination rule and the virtual service. Go to the destination rule and select outlier detection. Since Kiali only generates a simple template, let's adjust it. The field consecutiveErrors is just an old alias for consecutiveGatewayErrors. Let's replace it with 'consecutive5xxErrors' to treat all 5xx status codes as a circuit breaker indicator. Add a 1-minute interval for consecutive errors, with a 1-minute base ejection time. We also set the maximum Ejection Percent to 100% to allow all failing pods to be ejected if necessary, ensuring traffic is not routed to unhealthy instances. In reality, always try to find a suitable match for these numbers on a case-by-case basis. Create it.

<img src="pics/kiali-yellow-service.png-3.png" width="1200" />
<br>
<br>

<img src="pics/kiali-yellow-service.png-4.png" width="1200" />
<br>
<br>

Or create with file devops-istio-circuit-braker-ready.yml

    CMD --> kubectl apply -f devops-istio-circuit-braker-ready.yml

    # result:
    destinationrule.networking.istio.io/devops-yellow-clusterip created
    virtualservice.networking.istio.io/devops-yellow-clusterip created

Go to the Lens and open the log in the yellow container.

<img src="pics/lens-yellow-logs.png" width="1200" />
<br>
<br>

<img src="pics/lens-yellow-logs-1.png" width="1200" />
<br>
<br>

Make sure minikube tunnel is up

    CMD --> minikube tunnel

Now open Postman and run chain call 4, which calls yellow. Add 2xx or even 4xx as a parameter. 

<img src="pics/postman-2xx-4th-call.png" width="1200" />
<br>
<br>

Run it 300 times with a 2-second interval.

<img src="pics/postman-run-4th.png" width="1200" />
<br>
<br>

See the log on Lens. Approximately every 2 seconds, we log an entry.

<img src="pics/lens-yellow-logs-2.png" width="1200" />
<br>
<br>

Stop the postman collection runner.

<img src="pics/postman-stop-run.png" width="1200" />
<br>
<br>

Re-run 'chain call 4'. This time, use 5xx as a parameter. 

<img src="pics/postman-5xx-4th-call.png" width="1200" />
<br>
<br>

Run it 300 times with a 2-second interval.

<img src="pics/postman-run-4th.png" width="1200" />
<br>
<br>

See the log on Lens. Approximately every 2 seconds, we have a log entry. But after 5 attempts, with 5xx status, the log stops. This log shows that the circuit breaker is triggered and that no traffic is flowing to the pod, indicating back pressure on the pod.

<img src="pics/lens-yellow-logs-3.png" width="1200" />
<br>
<br>

During this circuit breaker trigger period, even if we send a 2xx response, it will still not reach the service. 

<img src="pics/poastman-5xx-circuit-result.png" width="1200" />
<br>
<br>

Note that the client (the blue in this case) still gets an error and must handle it itself. Since yellow had no traffic, yellow had time to process anything that might still be pending on yellow's side. Thus, if it's just an intermittent error due to spiking traffic or busy resources, yellow might recover on its own. Also, blue will only need a small response time to receive an error, since the circuit breaker has already terminated the traffic. Also, notice in the yellow log: since we set the base ejection time to 1 minute, after 1 minute of inactivity, the circuit breaker will be cancelled, and traffic will resume.

<img src="pics/lens-yellow-logs-4.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)