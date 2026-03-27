# Install Minikube setup of microservice application architecture with most required features

## Content

- [Install Minikube setup of microservice application architecture with most required features](#install-minikube-setup-of-microservice-application-architecture-with-most-required-features-1)       
[77 Install Kube Prometheus](#77-install-kube-prometheus)       
[78 Install Ingress Controller](#78-install-ingress-controller)     
[80 Install Istio](#80-install-istio)     
[77 Install Kube Prometheus](#77-install-kube-prometheus-2)     
[77 Install Kube Prometheus](#77-install-kube-prometheus-3)     


## Install Minikube setup of microservice application architecture with most required features

Delete the previous minikube and start fresh Minikube cluster

    bash --> minikube delete
    bash --> minikube start --cpus 4 --memory 16384 --driver docker

List contexts

    bash --> kubectl config get-contexts

Set minikube contexts

    bash --> kubectl config use-context minikube

Start minikube tunnel and don't close the terminal

    bash --> minikube tunnel

Make sure that address are added to Windows host list

- Open PowerShell as Admin

        PowerShell --> notepad C:\Windows\System32\drivers\etc\hosts

- add

```text
127.0.0.1 localhost                     
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

## 77 Install Kube Prometheus

[⬆ Back to top](#top)

Install Kube Prometheus Stack using Helm:

    CMD --> helm upgrade --install kube-prometheus-stack --repo https://prometheus-community.github.io/helm-charts kube-prometheus-stack --namespace istio-system --create-namespace -f values-kube-prometheus.yml

    # result:
    Release "kube-prometheus-stack" does not exist. Installing it now.
    NAME: kube-prometheus-stack
    LAST DEPLOYED: Sat Mar 21 21:18:04 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:

[⬆ Back to top](#top)

## 78 Install Ingress Controller

[⬆ Back to top](#top)

Add haproxy helm repo    

    CMD --> helm repo add haproxytech https://haproxytech.github.io/helm-charts
    CMD --> helm repo update

Install HAProxy using Helm

    CMD --> helm upgrade --install haproxy-ingress haproxytech/kubernetes-ingress --namespace haproxy --create-namespace --set controller.service.type=LoadBalancer --set controller.ingressClass=haproxy -f .\values-ingress-haproxy.yml

[⬆ Back to top](#top)

## 80 Install Istio

[⬆ Back to top](#top)

Install Istio custom resource definition - CRD

    CMD --> helm upgrade --install istio-base base --repo https://istio-release.storage.googleapis.com/charts --namespace istio-system --create-namespace

    # result:
    Release "istio-base" does not exist. Installing it now.
    NAME: istio-base
    LAST DEPLOYED: Fri Mar 27 16:56:42 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    Istio base successfully installed!

    To learn more about the release, try:
    $ helm status istio-base -n istio-system
    $ helm get all istio-base -n istio-system

Install Istio CNI (Container Network Interface) - daemon

    CMD --> helm upgrade --install istio-cni cni --repo https://istio-release.storage.googleapis.com/charts --namespace istio-system --create-namespace

    # result:
    Release "istio-cni" does not exist. Installing it now.
    I0327 16:58:57.992049   13452 warnings.go:107] "Warning: spec.template.metadata.annotations[container.apparmor.security.beta.kubernetes.io/install-cni]: deprecated since v1.30; use the \"appArmorProfile\" field instead"
    NAME: istio-cni
    LAST DEPLOYED: Fri Mar 27 16:58:57 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    "istio-cni" successfully installed!

    To learn more about the release, try:
      $ helm status istio-cni -n istio-system
      $ helm get all istio-cni -n istio-system

Then we will install istiod (control plane)

    CMD --> helm upgrade --install istiod istiod --repo https://istio-release.storage.googleapis.com/charts --namespace istio-system

    # result:
    Release "istiod" does not exist. Installing it now.
    NAME: istiod
    LAST DEPLOYED: Fri Mar 27 18:08:31 2026
    NAMESPACE: istio-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    "istiod" successfully installed!

    To learn more about the release, try:
      $ helm status istiod -n istio-system
      $ helm get all istiod -n istio-system

    Next steps:
      * Deploy a Gateway: https://istio.io/latest/docs/setup/additional-setup/gateway/
      * Try out our tasks to get started on common configurations:
        * https://istio.io/latest/docs/tasks/traffic-management
        * https://istio.io/latest/docs/tasks/security/
        * https://istio.io/latest/docs/tasks/policy-enforcement/
      * Review the list of actively supported releases, CVE publications and our hardening guide:
        * https://istio.io/latest/docs/releases/supported-releases/
        * https://istio.io/latest/news/security/
        * https://istio.io/latest/docs/ops/best-practices/security/

    For further documentation see https://istio.io website

Wait a while and check that we have 'istio-cni-node-xxxxx' and 'istiod-xxxxxxxxx-xxxxx' pod in the istio-system namespace.

    CMD --> kubectl get pods -n istio-system

    # result:
    NAME                                                        READY   STATUS    RESTARTS        AGE
    alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   2 (5d19h ago)   5d20h
    istio-cni-node-j8bjx                                        1/1     Running   0               70m
    istiod-6dbf7db644-2hgv8                                     1/1     Running   0               32s
    kube-prometheus-stack-grafana-95cccf84-p959z                3/3     Running   3 (5d19h ago)   5d20h
    kube-prometheus-stack-kube-state-metrics-844c755cdf-c5r7k   1/1     Running   1 (5d19h ago)   5d20h
    kube-prometheus-stack-operator-bd9dfdfd8-9mmsf              1/1     Running   1 (5d19h ago)   5d20h
    kube-prometheus-stack-prometheus-node-exporter-5hkks        1/1     Running   1 (5d19h ago)   5d20h
    prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   2 (5d19h ago)   5d20h

Install Istio scrape

    CMD --> kubectl apply -f istio-scrape.yml

    # result:
    podmonitor.monitoring.coreos.com/istio-sidecar-monitor created
    servicemonitor.monitoring.coreos.com/istio-component-monitor created

Disable Istion on Ingress Controller

    CMD --> kubectl apply -f disable-istio-on-ingress.yml

    # result:
    Warning: resource namespaces/haproxy is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
    namespace/haproxy configured

Restart Deploment

    CMD --> kubectl rollout restart deployment -n haproxy

    # result: deployment.apps/haproxy-ingress-kubernetes-ingress restarted

Deploy Application

    CMD --> kubectl apply -f devops-istio-basic-deployment-2.0.0.yml

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

Enable sidecar injection in 'devops' namespace

    CMD --> kubectl apply -f devops-istio-enable-sidecar.yml

    # result: namespace/devops configured

Restart application deployments

    CMD --> kubectl rollout restart deployment -n devops

    # result:
    deployment.apps/istio-basic-deployment-blue restarted
    deployment.apps/istio-basic-deployment-white restarted
    deployment.apps/istio-basic-deployment-yellow restarted

Check that sidecar pods are deployed as well

    CMD --> kubectl get pods -n devops

    # result:
    NAME                                             READY   STATUS    RESTARTS   AGE
    istio-basic-deployment-blue-858894d5f6-rsz7m    2/2     Running   0           75s
    istio-basic-deployment-white-5867c888c5-znpmg   2/2     Running   0           75s
    istio-basic-deployment-yellow-8985dc9d9-kjk99   2/2     Running   0           75s

Start minikube tunnel

    CMD --> minikube tunnel

Access the ingress on the path http://localhost/prometheus/query and verify that there are metrics.

[⬆ Back to top](#top)

## 77 Install Kube Prometheus ]

[⬆ Back to top](#top)

[⬆ Back to top](#top)

## 77 Install Kube Prometheus

[⬆ Back to top](#top)

[⬆ Back to top](#top)