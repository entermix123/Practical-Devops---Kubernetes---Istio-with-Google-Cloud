# Section 7 Kubernetes User Interface

## Content
- 29 [Dashboard](#29-dashboard)
- 30 [Lens Dashboard](#30-lens-dashboard)

Start minikube tunnel and don't close the terminal

    bash --> minikube tunnel

## 29 Dashboard
[⬆ Back to top](#top)

Kubernetes also provides a user interface dashboard. When using a cloud provider, they will provide their own dashboard. When using minikube, we can enable it.

Minikube providesmany additional objects to use, like this. 

    bash --> minikube addons list

    # result:
    ┌─────────────────────────────┬──────────┬────────────┬────────────────────────────────────────┐
    │         ADDON NAME          │ PROFILE  │   STATUS   │               MAINTAINER               │
    ├─────────────────────────────┼──────────┼────────────┼────────────────────────────────────────┤
    │ ambassador                  │ minikube │ disabled   │ 3rd party (Ambassador)                 │
    │ amd-gpu-device-plugin       │ minikube │ disabled   │ 3rd party (AMD)                        │
    │ auto-pause                  │ minikube │ disabled   │ minikube                               │
    │ cloud-spanner               │ minikube │ disabled   │ Google                                 │
    │ csi-hostpath-driver         │ minikube │ disabled   │ Kubernetes                             │
    │ dashboard                   │ minikube │ disabled   │ Kubernetes          # dashboard        │    
    │ default-storageclass        │ minikube │ enabled ✅ │ Kubernetes                             │
    │ efk                         │ minikube │ disabled   │ 3rd party (Elastic)                    │
    │ freshpod                    │ minikube │ disabled   │ Google                                 │
    │ gcp-auth                    │ minikube │ disabled   │ Google                                 │
    │ gvisor                      │ minikube │ disabled   │ minikube                               │
    │ headlamp                    │ minikube │ disabled   │ 3rd party (kinvolk.io)                 │
    │ inaccel                     │ minikube │ disabled   │ 3rd party (InAccel [info@inaccel.com]) │
    │ ingress                     │ minikube │ disabled   │ Kubernetes                             │
    │ ingress-dns                 │ minikube │ disabled   │ minikube                               │
    │ inspektor-gadget            │ minikube │ disabled   │ 3rd party (inspektor-gadget.io)        │
    │ istio                       │ minikube │ disabled   │ 3rd party (Istio)                      │
    │ istio-provisioner           │ minikube │ disabled   │ 3rd party (Istio)                      │
    │ kong                        │ minikube │ disabled   │ 3rd party (Kong HQ)                    │
    │ kubeflow                    │ minikube │ disabled   │ 3rd party                              │
    │ kubetail                    │ minikube │ disabled   │ 3rd party (kubetail.com)               │
    │ kubevirt                    │ minikube │ disabled   │ 3rd party (KubeVirt)                   │
    │ logviewer                   │ minikube │ disabled   │ 3rd party (unknown)                    │
    │ metallb                     │ minikube │ disabled   │ 3rd party (MetalLB)                    │
    │ metrics-server              │ minikube │ disabled   │ Kubernetes                             │
    │ nvidia-device-plugin        │ minikube │ disabled   │ 3rd party (NVIDIA)                     │
    │ nvidia-driver-installer     │ minikube │ disabled   │ 3rd party (NVIDIA)                     │
    │ nvidia-gpu-device-plugin    │ minikube │ disabled   │ 3rd party (NVIDIA)                     │
    │ olm                         │ minikube │ disabled   │ 3rd party (Operator Framework)         │
    │ pod-security-policy         │ minikube │ disabled   │ 3rd party (unknown)                    │
    │ portainer                   │ minikube │ disabled   │ 3rd party (Portainer.io)               │
    │ registry                    │ minikube │ disabled   │ minikube                               │
    │ registry-aliases            │ minikube │ disabled   │ 3rd party (unknown)                    │
    │ registry-creds              │ minikube │ disabled   │ 3rd party (UPMC Enterprises)           │
    │ storage-provisioner         │ minikube │ enabled ✅ │ minikube                               │
    │ storage-provisioner-rancher │ minikube │ disabled   │ 3rd party (Rancher)                    │
    │ volcano                     │ minikube │ disabled   │ third-party (volcano)                  │
    │ volumesnapshots             │ minikube │ disabled   │ Kubernetes                             │
    │ yakd                        │ minikube │ disabled   │ 3rd party (marcnuri.com)               │
    └─────────────────────────────┴──────────┴────────────┴────────────────────────────────────────┘

Dashboard is one of them. To usethe dashboard, type this.

    bash --> minikube dashboard

Minikube will enable the dashboard addon and open the plugin in a web browser.

Now we can navigate all around. Choose 'devops' namespace from the top dropdown menu and check all resources.

We can even go inside the pod to see the log, for example.

Go to Pods menu and click details (3 dots) button for specific pod and choose logs 

[⬆ Back to top](#top)

## 30 Lens Dashboard
[⬆ Back to top](#top)

A dashboard alternative is Kubernetes Lens. Download it from this site - https://lenshq.io/. At the time this video was recorded, it was free. Try it: download, install, and log in. There's nothing special about the installation process, so I will skip that part. I will show you around, but the user interface when you purchase the course might be different from the one in this video. It should be fine.

This is the lens user interface. It is intuitive and easy to navigate. We can browse each Kubernetes object deployed on our cluster. If we have multiple Kubernetes clusters, Lens will detect them, and we can select which cluster to observe.

For navigating Kubernetes objects in this course, feel free to use Lens. The course will mostly use the kubectl command because it is more general. For complex navigation, the course might use a lens, as it is more user-friendly.


[⬆ Back to top](#top)