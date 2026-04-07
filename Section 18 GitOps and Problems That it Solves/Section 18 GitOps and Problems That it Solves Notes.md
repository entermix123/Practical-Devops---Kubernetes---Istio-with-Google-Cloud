# Section 18 GitOps and Problems That it Solves

## Content
- 63 [Loan Pig Bottleneck](#63-loan-pig-bottleneck)
- 64 [Introducing GitOps](#64-introducing-gitops)
- 65 [GitOp with ArgoCD](#65-gitop-with-argocd)
- 66 [ArgoCD Synchronization](#66-argocd-synchronization)
- 67 [Kubernetes CRD (Custom Resource Definition)](#67-kubernetes-crd-custom-resource-definition)
- 68 [Loan Pig Bottleneck Solution](#68-loan-pig-bottleneck-solution)
- 69 [Tips: ArgoCD Useful Features](#69-tips-argocd-useful-features)
- 70 [ArgoCD Sensitive Data](#70-argocd-sensitive-data)
- 71 [Reloader](#71-reloader)

Delete the previous minikube and start fresh Minikube cluster

    bash --> minikube delete
    bash --> minikube start --cpus 4 --memory 8192 --driver docker

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

## 63 Loan Pig Bottleneck

[⬆ Back to top](#top)

Let's get back to the fiction company (loan pig) at the early stage of this course. At this point, the loan pig has a DevOps team that deploys the application to a Kubernetes cluster. They are quite handy with all of those YAML and tools. Let's review the technical steps in the DevOps cycle. 

After the engineering team writes code implementation The engineer compiled it, and the engineering team tests it. Note that I say \"engineering team\", which may consist of a software engineer and a quality-assurance engineer. The application, which has passed testing, is then built into a Docker image, and the engineering team pushes it to the company's Docker repository. But not just a Docker image; sometimes environment variables stored as config maps or secrets must also be added, removed, or changed. Assume that only the DevOps team has access to the Kubernetes deployment. So, engineering sends a message to the DevOps team, telling them that a new image has been pushed and needs to be deployed. The DevOps team released the application as a Helm chart, so they need to update the Helm values to the latest image, then redeploy the Helm chart. 

<img src="pics/software-development-phase.png" width="700" />
<br>
<br>

Let's focus on this deployment phase, which marks a transition between the engineering and DevOps teams.

<img src="pics/devops-phase.png" width="700" />
<br>
<br>

Apparently, the CTO heard complaints that the DevOps team's deployment process is slow and has become a bottleneck. After a lunch session with the DevOps team, the CTO has come to understand several things. 
- The DevOps team manages 3 environments for the company: development, testing, and production. 
- The DevOps engineer's task is not just deployment; it also includes a range of monitoring, development, improvement, and operational work. 

There are more than 10 squads, some of which release one new Docker image each day in the dev and test environments. But there are only 3 people in the DevOps team. Some squads also operate in different time zones, so they need to deploy when all DevOps engineers are asleep. So the CTO asks whether adding new people will solve the problem. But the DevOps team does not really agree with this solution. Adding more people will only delay such a problem. And what about that team in a different timezone? It is not efficient for the DevOps team to be ready 24 hours a day to deploy the application. After all, DevOps is about automation. The loan pig is already one step closer to automatic deployment using Kubernetes. 

<img src="pics/devops-bottleneck.png" width="800" />
<br>
<br>

Let's take another step and figure out how to automate the deployment process so that when someone in engineering releases a Docker image, the deployment happens automatically. Or at least let the engineering team trigger deployments themselves, without waiting for the DevOps engineer. One thing to consider is that the engineering team must not have access to the Kubernetes cluster, but they can still trigger deployments.

In real life, the process of building and pushing a Docker image can be automated. This process is what we call the CI pipeline, or continuous integration. This automatic deployment is called the CD pipeline, or continuous deployment. But since this course focuses on Kubernetes, which heavily involves deployment, let's focus on that. The alternative solution for this problem is in the next lesson.

<img src="pics/cd-continues-deplyment.png" width="700" />
<br>
<br>

[⬆ Back to top](#top)


## 64 Introducing GitOps

[⬆ Back to top](#top)

The DevOps team proposes an alternative solution. Everybody on the engineering team is familiar with Git. So what if they have a solution based on Git? The idea is to put a chart and values into Git, as we learned in the previous lesson. Image name is already part of the values. So if the engineering team updates the image name or other predefined settings in values.yaml, and that values file is pushed to GitHub, The DevOps team can pull the chart and values from GitHub and create a Helm release from GitHub.  

<img src="pics/gitops-the-idea.png" width="800" />
<br>
<br>

This course will not cover GitHub flow in detail, but the approach has benefits. Usually, there is also an established Git flow. There is a main branch to be deployed. A task is created via a GitHub issue or a Jira ticket. An engineer works on this task, Creating a new git branch for that issue. The engineer added some commits for that task on the new branch. Once the task is complete, a pull request is created. At this point, a continuous integration (CI) pipeline usually runs. A DevOps engineer usually builds a CI pipeline that includes tasks such as formatting source code, compiling it into a binary, running static analysis tools to find potential bugs or security holes, validating code quality, and running unit tests. If the CI pipeline runs successfully, the code is reviewed by the engineering team, and some discussion and changes may occur. This loop can go a few rounds until the code is approved. The approved code was then merged back into the main branch. At this point, the continuous delivery (CD) pipeline runs, with tasks that build a new Docker image with a new Docker tag, push the image to the repository, and then deploy the Docker on the server. This server can be as simple as a machine with Docker runtime or Kubernetes. When the application is deployed, then it's finished.     

<img src="pics/development-based-on-main-branch.png" width="800" />
<br>
<br>

DevOps teams usually focus on building CI/CD pipelines using tools such as Jenkins or GitHub Actions. For this Kubernetes course, and the problem that the loan pig is facing, Let's assume the loan pig team has not implemented a complete CI/CD pipeline, and the process is still manual. Right now, they have Kubernetes running, and the bottleneck problem occurs in the deployment phase. This can happen because the engineering team usually does not have access to the server, so the DevOps team needs to update the server's Docker image. A traditional DevOps approach usually uses a push-based approach, where tools like Jenkins or GitHub Actions deploy the new Docker image to the server.
<br>
<br>
<img src="pics/manual-devops-example.png" width="800" />
<br>
<br>
An alternative isalso available: GitOps. The application source code has its own git repository. In GitOps, we have another Git repository that can differ from the application source code repository. Let's call such a repository the application deployment repository. In GitOps, the CI/CD pipeline shown on the previous slide can run in the application source code repository, where a merge into the application main branch builds a Docker image, tags it, and pushes it to the Docker repository. But instead of deploying Docker to the server, the CD pipeline updates the application deployment repository with a new Docker image tag. We use the GitOps tool to monitor the git repository for this application deployment and the current state of the Kubernetes cluster. When a file is changed in a git repository, the status between the deployment on Kubernetes and the repository becomes different. This tool will then synchronize changes from the application deployment git repository with Kubernetes, using Git as the source of truth. For example, if the application deployment repository has updated values .YAML file, where the Docker image tag changed. Or the pod replica changed from 2 to 4, or an environment variable changed. The tool will then update Kubernetes and synchronize the changes from the GitHub repository into the Kubernetes state. Technically speaking, Kubernetes has an actual state, Git has a target state, and this GitOps tool will monitor the application deployment Git repository for changes to the target state. The main task of this GitOps tool is to synchronize the actual state with the target state. So even if someone accidentally deletes a pod in a Kubernetes cluster, this GitOps tool will know that the actual pod count is 0, while the target state should have 2 pods, so it will recreate 2 pods to match the actual and target states.
<br>
<br>
<img src="pics/gitops-problem-solve.png" width="800" />
<br>
<br>
This is the formula of full GitOps. We need three items: a Git pull request, a CI/CD pipeline, and infrastructure-as-code. Since all Kubernetes changes are tracked in Git, GitOps uses pull requests as the mechanism for all deployment updates. In a pull request, teams can collaborate through reviews and comments, and formal approvals are required. A pull approval and merge commit to the main branch can also be used as an audit log. 
<br>
<br>
<img src="pics/gitops-pr.png" width="800" />
<br>
<br>
GitOps uses continuous integration (CI) and continuous delivery (CD). When new code is introduced via a pull request or merge, the CI/CD pipeline triggers a deployment. Any configuration drift, such as manual changes or errors, is overwritten by GitOps automation, which moves the environment toward the target state defined in Git. 
<br>
<br>
<img src="pics/gitops-ci-cd.png" width="800" />
<br>
<br>
Infrastructure as code (IaC) is the technology for provisioning IT infrastructure using configuration files or code. This can be infrastructure on Kubernetes, like the number of replicas or ingress rules. 
be used as an audit log. Take it a step further, and we can provision cloud infrastructure such as SQL databases, file storage, or firewalls using code. This means we write declarative infrastructure as configuration files or code. For example, which SQL database product, and how many CPU cores, memory, and storage? All code can be stored in Git, and GitOps tools, rather than humans, will handle provisioning. Using IaC improves infrastructure provisioning consistency and reduces the need for human interaction to create cloud infrastructure.
<br>
<br>
<img src="pics/gitops-iac.png" width="800" />
<br>
<br>
This GitOps approach is what the loan pig devops team will use. So, in the sample case, the engineering team can update the values in the application deployment git repository with the latest Docker image tag and predefined configuration. The GitOps tool will automatically synchronize the change into the Kubernetes cluster. In real life, this update on the application deployment git repository can be automated with a CI pipeline. For this course, which focuses on Kubernetes, we will manually update the file.

[⬆ Back to top](#top)


## 65 GitOp with ArgoCD

[⬆ Back to top](#top)

The GitOps tool to be used is ArgoCD. We will install Argocd using Helm and configure the connection to the application deployment git repository. We will then deploy the first version of the application, update it to the second version, and watch how ArgoCD works. Please ensure the HAProxy ingress controller is installed, and the metrics-server addon is enabled. 

To go hands-on with this lesson, it will require some resources. We need at least 3 dedicated CPU cores, and 5 gigabytes of free memory allocated for minikube. If you're stuck on the lesson (for example, the pod cannot run), it might be because of a lack of resources. To allocate such a resource, make sure your laptop and Docker have the necessary memory and CPU, delete the existing minikube cluster, and start new minikube cluster.

Delete existing minikube cluster

    CMD --> minikube delete

Start minikube cluster with required resources

    CMD --> minikube start --cpus 4 --memory 8192 --driver docker

Install HAProxy with Helm

    CMD --> helm upgrade --install my-haproxy kubernetes-ingress --repo https://haproxytech.github.io/helm-charts --namespace haproxy --create-namespace --set controller.service.type=LoadBalancer

    # result:
    Release "my-haproxy" does not exist. Installing it now.
    NAME: my-haproxy
    LAST DEPLOYED: Tue Mar 17 10:06:27 2026
    NAMESPACE: haproxy
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    HAProxy Kubernetes Ingress Controller has been successfully installed.

    Controller image deployed is: "docker.io/haproxytech/kubernetes-ingress:3.2.6".
    Your controller is of a "Deployment" kind. Your controller service is running as a "LoadBalancer" type.
    RBAC authorization is enabled.
    Controller ingress.class is set to "haproxy" so make sure to use same annotation for
    Ingress resource.

    Service ports mapped are:
    - name: admin
        containerPort: 6060
        protocol: TCP
    - name: http
        containerPort: 8080
        protocol: TCP
    - name: https
        containerPort: 8443
        protocol: TCP
    - name: stat
        containerPort: 1024
        protocol: TCP
    - name: quic
        containerPort: 8443
        protocol: UDP

    Node IP can be found with:
    $ kubectl --namespace haproxy get nodes -o jsonpath="{.items[0].status.addresses[1].address}"

    The following ingress resource routes traffic to pods that match the following:
    * service name: web
    * client's Host header: webdemo.com
    * path begins with /

    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
        name: web-ingress
        namespace: default
        annotations:
        ingress.class: "haproxy"
    spec:
        rules:
        - host: webdemo.com
        http:
            paths:
            - path: /
            backend:
                serviceName: web
                servicePort: 80

    In case that you are using multi-ingress controller environment, make sure to use ingress.class annotation and match it
    with helm chart option controller.ingressClass.

    For more examples and up to date documentation, please visit:
    * Helm chart documentation: https://github.com/haproxytech/helm-charts/tree/main/kubernetes-ingress
    * Controller documentation: https://www.haproxy.com/documentation/kubernetes/latest/
    * Annotation reference: https://github.com/haproxytech/kubernetes-ingress/tree/master/documentation
    * Image parameters reference: https://github.com/haproxytech/kubernetes-ingress/blob/master/documentation/controller.md

Enable metrics-server on minikube:

    CMD --> minikube addons enable metrics-server

    # result:
    💡  metrics-server is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
    You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
        ▪ Using image registry.k8s.io/metrics-server/metrics-server:v0.8.1
    🌟  The 'metrics-server' addon is enabled

We need to install ArgoCD. We will install it using Helm. In the course resources, open the 'argocd 01' folder and the 'values.yaml' file. This file is the configuration for the ArgoCD installation we will use. The initial admin password will be password. 

argocd-01/values-argocd.yml

```yaml
configs:
  secret:
    # initial password : "password" (BCRYPT hash)
    argocdServerAdminPassword: $2a$12$n4pjYaCeLFGvZcUhQ3.TYOm2cwUhFXxcAeXTBFdhxtsTKhRMMFfcq
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    ingressClassName: haproxy
    hostname: argocd.local
    annotations:
      haproxy.org/server-proto: "h1"
      haproxy.org/ssl-redirect: "true"
    tls: true
    extraTls:
      - hosts:
          - argocd.local
        secretName: argocd-local-cert
    paths:
      - /
    pathType: Prefix
  resources:
    requests:
      cpu: "0.25"
      memory: 200M
    limits:
      cpu: "0.4"
      memory: 300M

repoServer:
  livenessProbe:
    initialDelaySeconds: 30
    timeoutSeconds: 5
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    initialDelaySeconds: 30
    timeoutSeconds: 5
    periodSeconds: 10
    failureThreshold: 3
  resources:
    requests:
      cpu: "0.1"
      memory: 128M
    limits:
      cpu: "0.5"
      memory: 400M      
```

###### return point 1

We will enable HTTPS with this setting, using a self-signed certificate. The argocd host we will use is argocd.local. Ensure you have already set it in the hosts file. Also, ensure you have a self-signed certificate for 'argocd.local' installed for the ingress, under the secret named 'argocd-local-cert'. For a refresher on how to install it, please see the lesson [Ingress over TLS](../Section%2010%20Exposing%20Kubernetes%20Pod/Section%2010%20Exposing%20Kubernetes%20Pod%20Notes.md#39-ingress-over-tls) in the earlier section of the course.

Generate a TLS certificate on https://regery.com/en/security/ssl-tools/self-signed-certificate-generator for 'argocd.local'. Download the public and private key pair. 

Create secret 'argocd-local-cert' with the certificates

    CMD --> kubectl create secret tls argocd-local-cert --key C:\Users\user_name\Downloads\argocd-local-privateKey.key --cert C:\Users\user_name\Downloads\argocd-local.crt         # set the correct username

    # result: secret/argocd-local-cert created

Open a terminal in this folder, and install it. As usual, you can copy and paste this command from the last lecture of the course.

    argocd-01 CMD --> helm upgrade --install my-argocd argo-cd --repo https://argoproj.github.io/argo-helm --namespace argocd --create-namespace --values values-argocd.yml

    # result:
    Release "my-argocd" does not exist. Installing it now.
    NAME: my-argocd
    LAST DEPLOYED: Wed Mar 18 11:15:29 2026
    NAMESPACE: argocd
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    In order to access the server UI you have the following options:

    1. kubectl port-forward service/my-argocd-server -n argocd 8080:443

        and then open the browser on http://localhost:8080 and accept the certificate

    2. enable ingress in the values file `server.ingress.enabled` and either
        - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
        - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


    After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

    (You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
    
Ensure minikube tunnel is running

    CMD --> minikube tunnel

Check it. Go to https://argocd.local/. If there is any invalid certificate message, ignore it. Log in using:
- username admin 
- password 'password'. 

<br>
<img src="pics/argocd-login-page.png" width="1200" />
<br>
<br>

Nice, we have ArgoCD running.

<img src="pics/argocd-ui.png" width="1200" />
<br>
<br>

Next, we need to create a git application deployment repository. This repository will contain chart metadata and values files. I will create a private repository on GitHub. This repository is usually private because it might contain values intended only for your organization. Add a blank README file.
<br>
<img src="pics/github-repo-create.png" width="600" />
<br>
<br>

Copy the repository address:

<br>
<img src="pics/github-clone-repo.png" width="800" />
<br>
<br>

Clone it into locally

    D:/repos CMD --> git clone https://github.com/entermix123/devops-app-deployment.git

    # result:
    Cloning into 'devops-app-deployment'...
    remote: Enumerating objects: 3, done.
    remote: Counting objects: 100% (3/3), done.
    remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
    Receiving objects: 100% (3/3), done.

###### return point 2

Then, in the argocd-02 folder, there is a subfolder named my-devops-app. If we look into that folder, we will find a Chart.yaml file that depends on the Spring Boot REST API chart, located in the GitHub Helm repository. This is the Spring Boot REST API from the previous lesson about using the [GitHub Helm repository](../Section%2017%20Creating%20and%20Using%20Helm%20Charts/Section%2017%20Creating%20nad%20Using%20Helm%20Charts%20Notes.md#61-github-as-helm-repository). If you haven't gone through that lesson, please finish it first. Adjust the repository URL with your own GitHub repo. 

Chart.yaml

```yaml
apiVersion: v2
name: my-devops-chart
type: application
version: 2.0.0
appVersion: 2.0.0
dependencies:
  - name: spring-boot-rest-api
    version: 0.1.0
    # Change with your own github repo (from lesson : Using github as helm repository)
    repository: https://entermix123.github.io/devops-helm-charts/
```

We have two value files: one for default values and one for the dev environment.

values.yml

```yaml
spring-boot-rest-api:
  applicationPort: 8111

  actuatorPort: 8111

  prometheusPath: /devops/blue/actuator/prometheus

  health:
    readiness:
      path: /devops/blue/actuator/health/readiness
      port: 8111
      scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 30
      timeoutSeconds: 5
      failureThreshold: 4
    liveness:
      path: /devops/blue/actuator/health/liveness
      port: 8111
      scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 30
      timeoutSeconds: 5
      failureThreshold: 4

  extraVolumeMounts: |
    - name: upload-image-empty-dir
      mountPath: /upload/image
    - name: upload-doc-empty-dir
      mountPath: /upload/doc

  extraVolumes: |
    - name: upload-image-empty-dir
      emptyDir: {}
    - name: upload-doc-empty-dir
      emptyDir: {}

  service:
    type: ClusterIP
    port: 80

  # -------------------------------------------------------
  # For gateway API
  # The gateway `devops-gateway` must already exists
  httpRoute:
    enabled: false

  ingress:
    enabled: true
    className: haproxy
```

values-dev.yaml

```yaml
spring-boot-rest-api:
  image:
    repository: timpamungkas/devops-blue
    tag: 2.0.0
    
  autoscaling:
    enabled: false
    targetCPUUtilizationPercentage: ""
    targetMemoryUtilizationPercentage: ""

  ingress:
    annotations: 
      haproxy.org/rate-limit-period: 1m
      haproxy.org/rate-limit-requests: "10"
      haproxy.org/rate-limit-status-code: "429"
    hosts:
    - host: api.devops.local
      paths:
      - path: /devops/blue
        pathType: Prefix
        
  extraEnv: |
    - name: DEVOPS_BLUE_HTML_BG_COLOR
      value: "#dcecf2"
    - name: DEVOPS_BLUE_HTML_TEXT_COLOR
      value: "#000000"
    - name: HARDCODED_ENVIRONMENT_VARIABLE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_ONE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_TWO
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_THREE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_FOUR
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_FIVE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_SIX
      value: This is from values-dev.yml
```

Add this entire folder, including the folder itself, into the application deployment repository.

devops-app-deployment
                |-- README.md
                |-- my-devops-app           # copy the folder into our main repository folder

Commit the changes to the main branch and push them to GitHub.

    devops-app-deployment CMD --> git add .

    devops-app-deployment CMD --> git commit -m 'devops course'

    # result:
    [main 2593ddc] devops course
    3 files changed, 100 insertions(+)
    create mode 100644 my-devops-app/Chart.yaml
    create mode 100644 my-devops-app/values-dev.yaml
    create mode 100644 my-devops-app/values.yaml

    devops-app-deployment CMD --> git push

    # result:
    Enumerating objects: 7, done.
    Counting objects: 100% (7/7), done.
    Delta compression using up to 12 threads
    Compressing objects: 100% (6/6), done.
    Writing objects: 100% (6/6), 1.36 KiB | 1.36 MiB/s, done.
    Total 6 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
    To https://github.com/entermix123/devops-app-deployment.git
      8f3d1c5..2593ddc  main -> main

The values-xxxx.yaml file contains a value for each different software development environment, but keeping the YAML structure the same. So the xxxx can be development, test, or production. We can later have the same key, but different values per environment. For example, database URLs, usernames, passwords, API keys, and other credentials will differ per environment. For values that will be the same across environments, we can put them in values.yaml. For example, an application port or a health check. This way, we can manage values more structurally.

So the application deployment repository structure will be like this.

<img src="pics/github-repo-file-structure.png" width="1600" />
<br>
<br>

Now we need to create a GitHub personal access token to connect ArgoCD with GitHub. This step is specific only if we are using a GitHub private repository. The links might change at times. Please follow the steps in the GitHub documentation to create this. For now, this is how we create the fine-grained access token. 

Go to Profile / Settings / Developer Settings

<img src="pics/github-profile-settings.png" width="300" />
<br>
<br>

<img src="pics/github-developer-settings.png" width="300" />
<br>
<br>

<img src="pics/github-fine-graned-token.png" width="600" />
<br>
<br>

<img src="pics/github-fine-graned-token-2.png" width="1000" />
<br>
<br>
Give it read-only access to the repository content.

<img src="pics/github-fine-graned-token-3.png" width="1000" />
<br>
<br>

<img src="pics/github-fine-graned-token-4.png" width="600" />
<br>
<br>
Copy this token value now, as we can only see it once. 

<img src="pics/github-fine-graned-token-5.png" width="1000" />
<br>
<br>
To connect ArgoCD to GitHub, open ArgoCD and navigate to 'Settings', then 'Repositories' in the ArgoCD UI. 

<img src="pics/argcd-setting-repos.png" width="1200" />
<br>
<br>

Click 'Connect Repo' and fill in the connection details. Connection method: choose 'via HTTPS'. Type: Select 'git'. Give it any name. Project: Choose the project (or 'default'). Repository URL: Enter the devops-app-deployment GitHub repository URL.  Username: Enter your GitHub username. Password: Paste your personal access token here.
<br>
<img src="pics/argocd-connect-repo.png" width="1200" />
<br>
<br>

Click 'connect' and wait a while. The connection should be successful.
<br>
<img src="pics/argocd-github-repo-connected.png" width="1400" />
<br>
<br>

Now go to the ArgoCD application and add one. Name it 'dev-my-devops-app'. Set the synchronization policy to automatic and enable self-heal, so ArgoCD will always monitor the git repository for configuration changes and update Kubernetes when necessary. I will also enable auto-create namespaces and apply out-of-sync only, but these are optional. 
<br>
<img src="pics/argocd-create-app-1.png" width="1200" />
<br>
<br>
For the repository URL, select the one we have just created. Use head revision, with path to my-devops-app. Select a Kubernetes cluster as the destination. We currently only have a local one. Use 'dev' as a namespace. 
<br>
<img src="pics/argocd-create-app-2.png" width="1200" />
<br>
<br>

Then we can select the value files to be used. This time, we will use values.yaml and values-dev.yaml. ArgoCD will detect the values and show the preview.
<br>
<img src="pics/argocd-create-app-3.png" width="1200" />
<br>
<br>

Create the application, and wait a while. The synchronization process is running.
<br>
<img src="pics/argocd-create-app-4.png" width="800" />
<br>
<br>
We can see the application details and the object hierarchy created. Each of these boxes is a Kubernetes object from the application deployment repository, such as a service, deployment, or pod. 
<br>
<img src="pics/argocd-create-app-5.png" width="1400" />
<br>
<br>

We can see the summary and the Kubernetes event for each object. In the pod, we can also view the pod log without having to go inside the Kubernetes cluster.
<br>
<img src="pics/argocd-create-app-6.png" width="1200" />
<br>
<br>

Behind the scenes, Argo CD creates Kubernetes objects.

    CMD --> kubectl get pod,svc,ingress -n dev

    # result:
    NAME                                                          READY   STATUS    RESTARTS   AGE
    pod/dev-my-devops-app-spring-boot-rest-api-695565d968-lqgpj   1/1     Running   0          4m41s

    NAME                                             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
    service/dev-my-devops-app-spring-boot-rest-api   ClusterIP   10.108.5.77   <none>        80/TCP    4m41s

    NAME                                                               CLASS     HOSTS              ADDRESS     PORTS   AGE
    ingress.networking.k8s.io/dev-my-devops-app-spring-boot-rest-api   haproxy   api.devops.local   127.0.0.1   80      4m41s

Ensure minikube tunnel is running

    CMD --> minikube tunnel

Since we include the ingress creation, we can access the blue application as usual.
<br>
<img src="pics/postman-argocd-access.png" width="1400" />
<br>
<br>


[⬆ Back to top](#top)


## 66 ArgoCD Synchronization

[⬆ Back to top](#top)

What if someone accidentally changes the configuration, or worse, deletes a Kubernetes object monitored by Argo CD? Let's try it. I will use lens for operation, just for convenience. You can use kubectl if you want. I will delete the deployment from the dev namespace. This is the object that currently monitored by Argo CD. Thus, deleting it will create a difference between the actual Kubernetes state and the git targetstate Also, I will change the ingress flag and increase its rate limit. This step, of course, will also create a difference between the actual and the target state.
<br>
<img src="pics/lens-delete-deployment.png" width="1200" />
<br>
<br>
<img src="pics/rate-limit-edit-1.png" width="1200" />
<br>
<br>
<img src="pics/rate-limit-edit-2.png" width="1200" />
<br>
<br>

Wait a while and check Argo CD. 

It will synchronize the actual towards the target state. Therefore, we will have a new deploymenthere. The ingress rate limit returned to 10, which is the target state in git.
<br>
<img src="pics/argocd-sync-example-1.png" width="1200" />
<br>
<br>


If we check again from the lens, we will see a new deployment, and the ingressrate limit will return to the target state. 
<br>
<img src="pics/lens-new-deployment.png" width="1200" />
<br>
<br>

<br>
<img src="pics/lens-ingress-rate-limit-restore.png" width="1200" />
<br>
<br>
The case will be different if I turn off auto-sync in Argo CD. 
<br>
<img src="pics/argocd-disable-autosync.png" width="1000" />
<br>
<br>


Now, if I delete the deployment and change the ingress rate limit, Argo CD will tell us that the application is not synchronized, but it does nothing.
<br>
<img src="pics/lens-delete-deployment-2.png" width="1200" />
<br>
<br>
<img src="pics/rate-limit-edit-3.png" width="1200" />
<br>
<br>
<img src="pics/rate-limit-edit-3.png" width="1200" />
<br>
<br>
<img src="pics/rate-limit-edit-4.png" width="1200" />
<br>
<br>

ArgoCD out of sync:     
<br>
<img src="pics/argocd-out-of-sync.png" width="1200" />
<br>
<br>


We can still sync the state manually by using this button. It is up to you whether to use automatic or manual synchronization. 
<br>
<img src="pics/argocd-manual-sync.png" width="1200" />
<br>
<br>

Wait a while, and the state will be synchronized again. 
<br>
<img src="pics/argocd-resynced.png" width="1200" />
<br>
<br>

For now, let's delete the Argo CD application and the repository. We will learn something else.
<br>
<img src="pics/argocd-delete-app.png" width="1200" />
<br>
<br>
<img src="pics/argocd-disconnect-repo.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)


## 67 Kubernetes CRD (Custom Resource Definition)

[⬆ Back to top](#top)

Kubernetes has many objects, as we have already seen. Deployment, pod, ingress, horizontal pod autoscaler, etc. There are so many built-in resource definitions. But sometimes an application requires a unique resource to meet a custom requirement. In this case, the application can create a CRD (custom resource definition) for use.

Take Argo CD, for example. In the previous lesson, we created a repository credential and an application using the Argo CD user interface. Argo CD provides custom resource definitions, so we can create those things declaratively using a YAML file in the Argo CD CRD format. There are many Argo CD CRDs defined, so we can use them as needed. Remember, using declarative objects tells Kubernetes "what we need", and it creates them for us without human interference, which can lead to further automation. Browse the Argo CD CRD list and see what we need - https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/. 

For example, we will create my-devops-app from the previous lesson, using a declarative way. Open folder Argo CD 03, where we will find several files - argocd-a-repo-creds.yml, argocd-b-repository.yml and argocd-c-application.yaml. These files must run in order. Notice that all files must be in the Argo CD namespace. 

- First, the repository credential file, a Kubernetes secret with specific labels. Here we define the git url, username, and password for that git repo. Change the values to your own. 

argocd-a-repo-creds.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-argocd-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  # Change url, username, password with your own
  url: https://github.com/entermix123/devops-app-deployment
  username: entermix123
  password: github_pat_xxxxx
```
- Second, the repository, which indicates the application deployment repository to be monitored. 

argocd-b-repository.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-deployment-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # Change with your own
  url: https://github.com/entermix123/devops-app-deployment
```

- And third, the application. Here we define the Argo CD application behaviour, including self-heal or other synchronization policy.

argocd-c-application.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-my-devops-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/entermix/devops-app-deployment'
    path: my-devops-app
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

Run all three files in order.

    CMD --> kubectl apply -f argocd-a-repo-creds.yml

    # result:secret/my-argocd-repo-creds created
    
    CMD --> kubectl apply -f argocd-b-repository.yml

    # result: secret/my-app-deployment-repo created
    
    CMD --> kubectl apply -f argocd-c-application.yaml

    # result: application.argoproj.io/dev-my-devops-app created

Check the Argo CD. We should have a repository connectedand one application to synchronize.     
<br>
<img src="pics/arocd-added-repo-declarative.png" width="1400" />
<br>
<br>
<img src="pics/arocd-app-declarative.png" width="1200" />
<br>
<br>

[⬆ Back to top](#top)


## 68 Loan Pig Bottleneck Solution

[⬆ Back to top](#top)

Will using Argo CD solve the loan pig bottleneck? Why not. 

For example, an engineering team runs a process, pushes a new Docker image tag to the Docker repository, and someone updates the image tag on values-dev to the new version. Remember, in real life, we can implement a more robust process and automation, but for now, let's stick to the result: someone updates the application deploymentrepository.

Try it. Change the image tag in the application deployment repository to 2.0.1. Change it locally and push to git:

my-devops-app/values-dev.yml

```yaml
spring-boot-rest-api:
  image:
    repository: timpamungkas/devops-blue
    tag: 2.0.1                              # changed from 2.0.0
    ...
```

    CMD --> git add .
    CMD --> git commit -m 'v2.0.1'
    CMD --> git push    

Wait a while until Argo CD picks up the difference, and now synchronizing thechanges Wait a while until the pod is ready

CMD --> kubectl get pods -n dev

Check that the image tag is now 2.0.1.
<br>
<img src="pics/argocd-updated-app-version.png" width="1400" />
<br>
<br>

Make sure minikube tunnel is up

    bash --> minikube tunnel

Test with Postmen request:
<br>
<img src="pics/postman-access-app-v2-0-1.png" width="1400" />
<br>
<br>


[⬆ Back to top](#top)


## 69 Tips: ArgoCD Useful Features

[⬆ Back to top](#top)

In this lesson, we will see some Argo CD features that might be useful during operation. However, we will not implement them in this course, since they are not a really mandatory feature. The Argo CD documentation is quite complete, so feel free to experiment with the features. 

First, the developer can view the pod log (the application log) in Argo CD. They can access from this menu on each pod. 
<br>
<img src="pics/argocd-pod-logs.png" width="1400" />
<br>
<br>

Argo CD can also send notifications when an application goes down or becomes unhealthy, and when it comes back up - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/. There are many channels to send notifications, as simple as email, Slack, Microsoft Teams, etc.    
- google chat - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/googlechat/#google-chat.    
- email - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/email/    
- slack - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/slack/    
- teams workflows - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/teams-workflows/    

We can also create our own webhook if required - https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/webhook/

Argo CD is quite powerful, since we can restart or even delete an application from it. For this, Argo CD provides a user management module - https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/. 

There are many ways we can configure user management, but the one I'd like to inform you about is single sign-on (SSO) - https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#dex. 

For example, most developers will have a GitHub account and need to see whether the application is already deployed or check the application log. Instead of creating an Argo CD username for each developer, we can use Dex SSO in Argo CD so users can log in with their GitHub accounts. We can give read-only access to those GitHub accounts.

[⬆ Back to top](#top)


## 70 ArgoCD Sensitive Data

[⬆ Back to top](#top)

ArgoCD seems nice, but what if we need to work with sensitive data?

For example, what if we need to create secrets and monitor them with ArgoCD, To ensure that every change to the secret is reloaded automatically?

Well, we already learn sealed secret, which is a safe way to encrypt Kubernetes secrets, and the encryption can be put on a GitHub repository, which is monitored by ArgoCD. In this lesson, we will create a Helm chart for Sealed Secret, So ArgoCD can monitor it for any change. The chart template can be downloaded from the course resources and references, the last section of the course, in the folder helm-chart. 

To follow this lesson, ensure you have finished the previous lessons about these items and have them configured. As usual, references for the syntax and helm files are available in the last section of the course, folder resources & references.

###### return point 3

Go to D:\repos\devops-helm-charts\charts folder - the repository we set in the [61 GitHub as Helm Repository](../Section%2017%20Creating%20and%20Using%20Helm%20Charts/Section%2017%20Creating%20nad%20Using%20Helm%20Charts%20Notes.md#61-github-as-helm-repository) and create a Helm chart. I will name it "sealed-secret",but feel free to use any name.

    D:\repos\devops-helm-charts\charts\ CMD --> helm create sealed-secret

    # result: Creating sealed-secret

- In folder templates, remove all yml files except _helpers.tpl. 
- Copy and paste the sealedsecret.yml file from the course resources (devops-kubernetes-resources-references\kubernetes-istio-scripts\helm-charts\sealed-secret\templates\sealedsecret.yml) and references into the D:\repos\devops-helm-charts\charts\templates folder.

D:\repos\devops-helm-charts\charts\sealed-secret\templates\
- _helpers.tpl
- sealedsecret.yml    

sealedsecret.yml

```yaml
{{- if .Values.sealedSecrets }}
{{- range $sealedKey, $sealedValue := .Values.sealedSecrets }}
---
kind: SealedSecret
apiVersion: bitnami.com/v1alpha1
metadata:
  name: {{ $sealedValue.name }}
{{- if $sealedValue.annotations }}
  annotations:
{{- range $sealedValue.annotations }}
    {{- . | toYaml | nindent 4 }}
{{- end }}
{{- end }}
spec:
  template:
    type: {{ $sealedValue.type }}
    metadata:
      name: {{ .name }}
{{- if $sealedValue.annotations }}
      annotations:
{{- range $sealedValue.annotations }}
        {{- . | toYaml | nindent 8 }}
{{- end }}
{{- end }}
  encryptedData:
    {{- range $key, $value := $sealedValue.values }}
    {{ $key }}: {{ $value }}
{{- end }}
{{- end }}
{{- end }}
```

Adjust the D:\repos\devops-helm-charts\charts\sealed-secret\Chart.yml file according to our needs.

Chart.yml

```yaml
apiVersion: v2
name: sealed-secret
description: A Helm chart for sealed secrets        # changed

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
# It is recommended to use it with quotes.
appVersion: "1.16.0"
```

Then push this chart to the GitHub repository.

    helm-charts\devops-helm-charts\charts CMD --> git add .

    # result:
    warning: in the working copy of 'charts/sealed-secret/.helmignore', LF will be replaced by CRLF the next time Git touches it
    warning: in the working copy of 'charts/sealed-secret/Chart.yaml', LF will be replaced by CRLF the next time Git touches it
    warning: in the working copy of 'charts/sealed-secret/templates/_helpers.tpl', LF will be replaced by CRLF the next time Git touches it
    warning: in the working copy of 'charts/sealed-secret/values.yaml', LF will be replaced by CRLF the next time Git touches it

    helm-charts\devops-helm-charts\charts CMD --> git commit -m 'added sealed secret'

    # result:
    [main d993765] added sealed secret
    5 files changed, 300 insertions(+)
    create mode 100644 charts/sealed-secret/.helmignore
    create mode 100644 charts/sealed-secret/Chart.yaml
    create mode 100644 charts/sealed-secret/templates/_helpers.tpl
    create mode 100644 charts/sealed-secret/templates/sealedsecret.yml
    create mode 100644 charts/sealed-secret/values.yaml

    helm-charts\devops-helm-charts\charts CMD --> git push

    # result:
    Enumerating objects: 12, done.
    Counting objects: 100% (12/12), done.
    Delta compression using up to 12 threads
    Compressing objects: 100% (10/10), done.
    Writing objects: 100% (10/10), 3.94 KiB | 1.97 MiB/s, done.
    Total 10 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
    remote: Resolving deltas: 100% (1/1), completed with 1 local object.
    To https://github.com/entermix123/devops-helm-charts.git
      a956af0..d993765  main -> main

At this point, you should already be using GitHub as a Helm repository (61 GitHub as Helm Repository). So check the release page, and it should now contain the sealed-secret Helm chart.
<br>
<img src="pics/sealed-secret-in-helm-charts-repo.png" width="1600" />
<br>
<br>

### Install Sealed Secret Controller

Add Helm repository

    CMD --> helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

    # result: "sealed-secrets" has been added to your repositories


    CMD --> helm repo update

    # result:
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "chartmuseum" chart repository
    ...Successfully got an update from the "sealed-secrets" chart repository    # added 
    ...Successfully got an update from the "haproxytech" chart repository
    ...Successfully got an update from the "harbor" chart repository
    ...Successfully got an update from the "external-secrets" chart repository
    ...Successfully got an update from the "argo" chart repository
    Update Complete. ⎈Happy Helming!⎈

Install selaed-secret-controller

    CMD --> helm install sealed-secret-controller sealed-secrets/sealed-secrets --namespace kube-system --set fullnameOverride=sealed-secret-controller

    # result:
    NAME: sealed-secret-controller
    LAST DEPLOYED: Wed Mar 18 18:50:35 2026
    NAMESPACE: kube-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    ** Please be patient while the chart is being deployed **

    You should now be able to create sealed secrets.

    1. Install the client-side tool (kubeseal) as explained in the docs below:

        https://github.com/bitnami-labs/sealed-secrets#installation-from-source

    2. Create a sealed secret file running the command below:

        kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --format yaml > mysealedsecret.[json|yaml]

    The file mysealedsecret.[json|yaml] is a commitable file.

    If you would rather not need access to the cluster to generate the sealed secret you can run:

        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --fetch-cert > mycert.pem

    to retrieve the public cert used for encryption and store it locally. You can then run 'kubeseal --cert mycert.pem' instead to use the local cert e.g.

        kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --format [json|yaml] --cert mycert.pem > mysealedsecret.[json|yaml]

    3. Apply the sealed secret

        kubectl create -f mysealedsecret.[json|yaml]

    Running 'kubectl get secret secret-name -o [json|yaml]' will show the decrypted secret that was generated from the sealed secret.

    Both the SealedSecret and generated Secret must have the same name and namespace.

Wait untill the controller pod is running
  
    CMD --> kubectl get pods -n kube-system | Select-String "sealed"

    # result:
    NAME                                       READY   STATUS    RESTARTS     AGE
    sealed-secret-controller-bb89bdb9d-jpqpv   1/1     Running   0            59s

Then go to the D:\repos\devops-app-deployment repository, or your repository that ArgoCD monitors.Create subfolder: D:\repos\devops-app-deployment\my-secret\   

Inside, add the Chart.yml file, with a dependency on the sealed-secret Helm chart from your GitHub repository.

Chart.yml

```yaml
apiVersion: v2
name: my-sealed-secret-chart
type: application
version: 2.0.0
appVersion: 2.0.0
dependencies:
  - name: sealed-secret
    version: 0.1.0
    # Change with your own github repo (from lesson : Using github as helm repository)
    repository: https://entermix123.github.io/devops-helm-charts/
```

We also have a values-dev.yml file that will contain the sealed secret items.

values-dev.yml

```yaml
sealed-secret:
  namespace: devops               # Important!
  sealedSecrets:
    - name: devops-secret-blue    # Important!
      annotations:
        - sealedsecrets.bitnami.com/namespace-wide: "true"    # Important!
      type: Opaque
      values:
        devops-blue-html-text-one: AgC...WAw==
        devops-blue-html-text-two: AgA...xSw=
```

Files can be found in folder 'argocd-sensitive-data\my-secret' in the section resources

Push the changes to github.

    CMD --> git add .
    CMD --> git commit -m 'sealed secret'
    CMD --> git push

It is common for us to have different secrets per namespace, or, in this sample, the secret named "devops-secret-blue" will be deployed to the devops namespace. Notice the important items : Namespace, name, and annotation.

values-dev.yml

```yaml
sealed-secret:
  namespace: devops               # Important!
  sealedSecrets:
    - name: devops-secret-blue    # Important!
      annotations:
        # this secret will be known to all pods in the same namespace
        - sealedsecrets.bitnami.com/namespace-wide: "true"  
      type: Opaque
      values:
        devops-blue-html-text-one: AgC...WAw==
        devops-blue-html-text-two: AgA...xSw=
```

The annotation indicates that this secret will be known to all pods in the same namespace. In here, we have two encrypted, sealed secrets that can only be decrypted by our Kubernetes cluster. So it is safe to put this file on git.

But how do we get this sealed secret content? Not really different from the previous sealed secret. Open the source folder argocd-secret. See file devops-secret-plain.yml. It has two secret data items, encoded in base64. Notice that the name, namespace, and annotation here are the same as those in the Helm chart values, because we will seal this secret and then add it to the Helm chart values.

Download and install kubeseal from here - https://github.com/bitnami-labs/sealed-secrets/releases
- Extract the kubeseal.exe from the archive
- Move kubeseal.exe to a folder that's on your PATH for example C:/Users/username/.bin
- Check if we can use kubeseal: CMD --> kubeseal --version
  
devops-secret-plain.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-secret-blue        # Important!
  namespace: devops               # Important!
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"  # Important!
data:
 devops-blue-html-text-one: U2VhbGVkIHNlY3JldCB0ZXh0IC0gYmx1ZSBvbmU=
 devops-blue-html-text-two: TGV0J3MgdXBkYXRlIHRoaXMgc2VhbGVkIHNlY3JldCBpbnRvIHNvbWV0aGluZyBlbHNl
  # devops-blue-html-text-one: SGVsbG8hIEkgYW0gdGhlIG1pZ2h0eSAoYnV0IGtpbmQpIEt1YmVybmV0ZXMhIA==
  # devops-blue-html-text-two: QXJlIHlvdSBhIHBhbmRhPw==
```

So, the next step is to download the sealed secret controller certificate. I will download to this ArgoCD secret folder. 

    argocd-secret CMD --> kubeseal --controller-name=sealed-secret-controller --controller-namespace=kube-system --fetch-cert > mycert.pem

    # result: argocd-secret\mycert.pem

mycert.pem

```yaml
-----BEGIN CERTIFICATE-----
MIIEzTCCArWgAwIBAgIRALyf7dUF5prXJFN4GxOtkZMwDQYJKoZIhvcNAQELBQAw
...
...
qtJ3yvehS4QWv2LCKgXL4XaTwXZpIUyq+uLx5As0kKWZ
-----END CERTIFICATE-----
```

Then seal this base64-encoded secret file using the secret key. I will output this as devops-secret-sealed.yml in the same folder. 

    argocd-secret CMD --> kubeseal --cert mycert.pem -o yaml < devops-secret-plain.yml > devops-secret-sealed.yml

    result:

    devops-secret-sealed.yml

    ```yaml
    ---
    apiVersion: bitnami.com/v1alpha1
    kind: SealedSecret
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      name: devops-secret-blue
      namespace: devops
    spec:
      encryptedData:
        devops-blue-html-text-one: AgA...rzdi                         # copy value
        devops-blue-html-text-two: AgB...Wi/                          # copy value
      template:
        metadata:
          annotations:
            sealedsecrets.bitnami.com/namespace-wide: "true"
          name: devops-secret-blue
          namespace: devops
    ```

When we view the output, we will see two keys sealed. These are the data that were copied and pasted into Helm values. 

### Important note: you must use and update the sealed secret helm value for your environment. You cannot just use values from this course, as they can only be decrypted on my Kubernetes cluster. 

Move the secrets from argocd-secret\devops-secret-sealed.yml to D:\repos\devops-app-deployment\my-secret\values-dev.yml - the repo that is monitored from our ArgoCD.

values-dev.yml

```yaml
sealed-secret:
  namespace: devops               # Important!
  sealedSecrets:
    - name: devops-secret-blue    # Important!
      annotations:
        - sealedsecrets.bitnami.com/namespace-wide: "true"    # Important!
      type: Opaque
      values:
        devops-blue-html-text-one: AgA...L4Q==                     # paste value
        devops-blue-html-text-two: AgB...ufM=                      # paste value
```

Push the changes

    CMD --> git add .
    CMD --> git commit -m 'secrets update'
    CMD --> git push

In the folder ArgoCD-secret, there are several YAML files - argocd-a-repo-creds.yml, argocd-b-repository.yml, argocd-c-secret.yaml and argocd-d-secret-app.yaml. The repo credentials and repository are used to configure the connection between ArgoCD and the repository, so please update those files with your own credentials and repository. 

If you already set this in the previous ArgoCD lesson, skip those two files.

argocd-a-repo-creds.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-argocd-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  # Change url, username, password with your own
  url: https://github.com/entermix123/devops-app-deployment
  username: entermix123
  password: github_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

argocd-b-repository.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-deployment-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # Change with your own
  url: https://github.com/entermix123/devops-app-deployment
```


Change the repo URL in the third file argocd-c-secret.yaml, to your own, then apply it.

argocd-c-secret.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-secret
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/entermix123/devops-app-deployment'
    path: my-secret
    targetRevision: HEAD
    helm:
      valueFiles:
        - values-dev.yml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: devops
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

    argocd-secret CMD --> kubectl apply -f argocd-c-secret.yaml

    # result: application.argoproj.io/my-secret created

Check that ArgoCD now recognizes the secret. 
<br>
<img src="pics/argocd-cluster-secret-recognised.png" width="1400" />
<br>
<br>

At this point, we should have a secret named devops-secret-blue in the devops namespace, containing two keys. 

    CMD --> kubectl get secrets -n devops

    # result:
    NAME                 TYPE     DATA   AGE
    devops-secret-blue   Opaque   2      3m50s

If there is any change to the secret, we only need to deploy the sealed secret file to Git, and Argo CD will pick it up.

Let's try it. In the devops-app-deployment repository, create a new folder named my-devops-secret-app. Inside, add three files: Chart.yml, values.yml, and values-devops.yml from argocd-sensitive-data\my-devops-secret-app folder.

D:/repos/devops-app-deployment\my-devops-secret-app
- Chart.yml
- values.yml
- values-devops.yml

See the chart.yml. This configuration will deploy the Spring Boot REST API Helm chart we obtained in the previous lesson. 

Chart.yml

```yaml
apiVersion: v2
name: my-devops-chart
type: application
version: 1.0.0
appVersion: "1.0.0"
dependencies:
  - name: spring-boot-rest-api
    version: 0.1.0
    repository: https://entermix123.github.io/devops-helm-charts
```

On values-dev.yaml, We use values for these two texts from the secret.

values-dev.yaml

```yaml
spring-boot-rest-api:
  image:
    repository: timpamungkas/devops-blue
    tag: 2.0.0
    
  autoscaling:
    enabled: false

  ingress:
    annotations: 
      haproxy.org/rate-limit-period: 1m
      haproxy.org/rate-limit-requests: "20"
      haproxy.org/rate-limit-status-code: "429"

  extraEnv: |
    - name: DEVOPS_BLUE_HTML_BG_COLOR
      value: "#dcecf2"
    - name: DEVOPS_BLUE_HTML_TEXT_COLOR
      value: "#000000"
    - name: HARDCODED_ENVIRONMENT_VARIABLE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_ONE
      valueFrom:
        secretKeyRef:
          name: devops-secret-blue
          key: devops-blue-html-text-one
    - name: DEVOPS_BLUE_HTML_TEXT_TWO
      valueFrom:
        secretKeyRef:
          name: devops-secret-blue
          key: devops-blue-html-text-two
    - name: DEVOPS_BLUE_HTML_TEXT_THREE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_FOUR
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_FIVE
      value: This is from values-dev.yml
    - name: DEVOPS_BLUE_HTML_TEXT_SIX
      value: This is from values-dev.yml

  httpRoute:
    enabled: false
```

These secrets are actually data from sealed secrets that are monitored by ArgoCD and decrypted as Kubernetes regular secrets. 

Make sure this folder and all YAML files are pushed to the repository, which we will monitor again from ArgoCD - D:\repos\devops-app-deployment

    CND --> git add .
    CND --> git commit -m 'added app'

    # result:
    [main 02ecd16] added app
    3 files changed, 97 insertions(+)
    create mode 100644 my-devops-app/my-devops-secret-app/Chart.yaml
    create mode 100644 my-devops-app/my-devops-secret-app/values-dev.yaml
    create mode 100644 my-devops-app/my-devops-secret-app/values.yaml

    CND --> git push

    # result:
    Enumerating objects: 9, done.
    Counting objects: 100% (9/9), done.
    Delta compression using up to 12 threads
    Compressing objects: 100% (7/7), done.
    Writing objects: 100% (7/7), 1.44 KiB | 739.00 KiB/s, done.
    Total 7 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
    To https://github.com/entermix123/devops-app-deployment.git
      3cc2a28..02ecd16  main -> main

Still on the argocd-secret folder, see the argocd-d-secret-app.yaml file. Adjust the repo URL to your own. This time, we will monitor the app that uses a secret. 

argocd-d-secret-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-secret-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/entermix123/devops-app-deployment'
    path: my-devops-secret-app
    targetRevision: HEAD
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: devops
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

Apply this file and make sure ArgoCD now has a new application to monitor.

    CMD --> kubectl apply -f argocd-d-secret-app.yaml

    # result: application.argoproj.io/my-secret-app created
    
Wait a while until the application is provisioned.

<img src="pics/argocd-secret-app-depoyment.png" width="1200" />
<br>
<br>

<img src="pics/argocd-secret-app-synced.png" width="1200" />
<br>
<br>

Now, cross-check the secret value. Notice the value. It is base64-encoded. 

    CMD --> kubectl get secret -n devops -o yaml devops-secret-blue

    # result:
```yaml
apiVersion: v1
data:
  devops-blue-html-text-one: SGVs...hIA==           # base64-encoded
  devops-blue-html-text-two: QXJ...hPw==            # base64-encoded
kind: Secret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  creationTimestamp: "2026-03-18T18:45:01Z"
  name: devops-secret-blue
  namespace: devops
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: devops-secret-blue
    uid: 16183129-633f-42ac-abfb-7a682c079fdb
  resourceVersion: "30385"
  uid: 8b53545d-7b8d-4707-baa2-9a225cc58ef1
type: Opaque
```

Ensure minikube tunnel is up

    CMD --> minikube tunnel

Access the application from the configmap secret, and verify that the first and second text values are pulling data from the secret.

Make request with Postman
<br>
<img src="pics/postman-secret-ap-access-1.png" width="1200" />
<br>
<br>

<img src="pics/postman-sealed-secret-1.png" width="1200" />
<br>
<br>




Go back to the devops-secret-plain.yml file, and change the value of the text. Change it to whatever you want. 

devops-secret-plain.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-secret-blue        # Important!
  namespace: devops               # Important!
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"  # Important!
data:
  # devops-blue-html-text-one: U2VhbGVkIHNlY3JldCB0ZXh0IC0gYmx1ZSBvbmU=
  # devops-blue-html-text-two: TGV0J3MgdXBkYXRlIHRoaXMgc2VhbGVkIHNlY3JldCBpbnRvIHNvbWV0aGluZyBlbHNl
  devops-blue-html-text-one: SGVsbG8hIEkgYW0gdGhlIG1pZ2h0eSAoYnV0IGtpbmQpIEt1YmVybmV0ZXMhIA==
  devops-blue-html-text-two: QXJlIHlvdSBhIHBhbmRhPw==
```

Seal the new value.

    argocd-secret CMD --> kubeseal --cert mycert.pem -o yaml < devops-secret-plain.yml > devops-secret-sealed-update.yml

    # result:

    devops-secret-sealed-update.yml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  name: devops-secret-blue
  namespace: devops
spec:
  encryptedData:
    devops-blue-html-text-one: AgA...rzdi                         # copy value
    devops-blue-html-text-two: AgB...Wi/                          # copy value
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      name: devops-secret-blue
      namespace: devops
```

So, the next step is to download the sealed secret controller certificate. I will download to this ArgoCD secret folder. 

```yaml
sealed-secret:
  namespace: devops               # Important!
  sealedSecrets:
    - name: devops-secret-blue    # Important!
      annotations:
        - sealedsecrets.bitnami.com/namespace-wide: "true"    # Important!
      type: Opaque
      values:
        devops-blue-html-text-one: AgA2x...PZd              # paste values
        devops-blue-html-text-two: AgAW...AbM               # paste values
```

Then push it to GitHub.

  CMD --> git add .   
  CMD --> git commit -m 'edited secret'   
  CMD --> git push    

Wait a few moments for ArgoCD to pick up the change, or we can synchronize the value manually to speed things up.

<br>
<img src="pics/argocd-secret-sync.png" width="1400" />
<br>
<br>

Then check the secret value. It's now automatically updated. And check on the application.

    CMD --> kubectl get secret -n devops -o yaml devops-secret-blue

    # result:

```yaml
apiVersion: v1
data:
  devops-blue-html-text-one: U2...bmU=                    # different secret
  devops-blue-html-text-two: TG...bHNl                    # different secret
kind: Secret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  creationTimestamp: "2026-03-18T18:45:01Z"
  name: devops-secret-blue
  namespace: devops
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: devops-secret-blue
    uid: 16183129-633f-42ac-abfb-7a682c079fdb
  resourceVersion: "33916"
  uid: 8b53545d-7b8d-4707-baa2-9a225cc58ef1
type: Opaque
```

Restart the deployment. 

<img src="pics/argocd-restart-deployment-manually.png" width="1200" />
<br>
<br>

<img src="pics/argocd-restart-deployment-manually-result.png" width="1200" />
<br>
<br>

Make sure minikube tunnel is up

    bash --> minikube tunnel

Make request with Postman

<img src="pics/postnam-changed-secrets.png" width="1200" />
<br>
<br>

And now it's changed into a new value, but I need to restart the application. 

This case is actually normal. Remember, we have two ArgoCD, each monitors a different git repository. The secret repo and the application repo that uses the secret. When we change the secret repo, ArgoCD will update the data on the Kubernetes cluster, or, to be exact, the secret. Unfortunately, changes to secrets or config maps will not be considered an application change. Hence, we need to restart the pod so it picks up the change. ArgoCD cannot help here, since the application deployment has not changed.

<img src="pics/secret-change-for-argocd.png" width="800" />
<br>
<br>


[⬆ Back to top](#top)


## 71 Reloader

[⬆ Back to top](#top)

In a certain way, Argo CD can detect changes to secrets, as we saw earlier. A similar approach can be implemented in a configmap. Try practicing creating one if you need to. However, the pod itself needs to be restarted to pick up the change to the secret or config map. To accommodate automatic pod reloading when such a change occurs, we can use a tool named reloader. 

In this lesson, we will see a simple case with a reloader. We will use a configmap and a sealed secret for the sample, but not using Argo CD. We can combine with Argo CD if needed. For the lesson, make sure you have a sealed secret installed.

### Install Sealed Secret Controller

Add Helm repository

    CMD --> helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

    # result: "sealed-secrets" has been added to your repositories


    CMD --> helm repo update

    # result:
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "chartmuseum" chart repository
    ...Successfully got an update from the "sealed-secrets" chart repository    # added 
    ...Successfully got an update from the "haproxytech" chart repository
    ...Successfully got an update from the "harbor" chart repository
    ...Successfully got an update from the "external-secrets" chart repository
    ...Successfully got an update from the "argo" chart repository
    Update Complete. ⎈Happy Helming!⎈

Install selaed-secret-controller

    CMD --> helm install sealed-secret-controller sealed-secrets/sealed-secrets --namespace kube-system --set fullnameOverride=sealed-secret-controller

    # result:
    NAME: sealed-secret-controller
    LAST DEPLOYED: Wed Mar 18 18:50:35 2026
    NAMESPACE: kube-system
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    ** Please be patient while the chart is being deployed **

    You should now be able to create sealed secrets.

    1. Install the client-side tool (kubeseal) as explained in the docs below:

        https://github.com/bitnami-labs/sealed-secrets#installation-from-source

    2. Create a sealed secret file running the command below:

        kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --format yaml > mysealedsecret.[json|yaml]

    The file mysealedsecret.[json|yaml] is a commitable file.

    If you would rather not need access to the cluster to generate the sealed secret you can run:

        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --fetch-cert > mycert.pem

    to retrieve the public cert used for encryption and store it locally. You can then run 'kubeseal --cert mycert.pem' instead to use the local cert e.g.

        kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
        kubeseal \
          --controller-name=sealed-secret-controller \
          --controller-namespace=kube-system \
          --format [json|yaml] --cert mycert.pem > mysealedsecret.[json|yaml]

    3. Apply the sealed secret

        kubectl create -f mysealedsecret.[json|yaml]

    Running 'kubectl get secret secret-name -o [json|yaml]' will show the decrypted secret that was generated from the sealed secret.

    Both the SealedSecret and generated Secret must have the same name and namespace.

Wait untill the controller pod is running
  
    CMD --> kubectl get pods -n kube-system | Select-String "sealed"

    # result:
    NAME                                       READY   STATUS    RESTARTS     AGE
    sealed-secret-controller-bb89bdb9d-jpqpv   1/1     Running   0            59s

Download and install kubeseal from here - https://github.com/bitnami-labs/sealed-secrets/releases
- Extract the kubeseal.exe from the archive
- Move kubeseal.exe to a folder that's on your PATH for example C:/Users/username/.bin
- Check if we can use kubeseal: CMD --> kubeseal --version


### Install Reloader

Install the reloader using Helm. 

    CMD --> helm upgrade --install my-reloader reloader --repo https://stakater.github.io/stakater-charts --namespace reloader --create-namespace

    # result:
    Release "my-reloader" does not exist. Installing it now.
    NAME: my-reloader
    LAST DEPLOYED: Wed Mar 18 22:45:36 2026
    NAMESPACE: reloader
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    NOTES:
    - For a `Deployment` called `foo` have a `ConfigMap` called `foo-configmap`. Then add this annotation to main metadata of your `Deployment`
      configmap.reloader.stakater.com/reload: "foo-configmap"

    - For a `Deployment` called `foo` have a `Secret` called `foo-secret`. Then add this annotation to main metadata of your `Deployment`
      secret.reloader.stakater.com/reload: "foo-secret"

    - After successful installation, your pods will get rolling updates when a change in data of configmap or secret will happen
  
Open the folder reloader, which contains several YAML files - devops-reloader-configmap.yml, devops-reloader-secret-plain.yml and devops-reloader-deployment.yml. 

devops-reloader-configmap.yml file is the configmap configuration. 

devops-reloader-configmap.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name:  devops

---

kind: ConfigMap 
apiVersion: v1 
metadata:
  namespace: devops
  name: devops-blue-configmap
immutable: false
data:
  # html.color.background: "#F7E396"
  # html.color.text: "#5C3E94"
  # html.text.one: "Hello there, I am the first configmap"
  # html.text.two: "Just another configmap (2nd)"
  # html.text.three: "This is the THIRD text from k8s configmap"
  html.color.background: "#0F828C"
  html.color.text: "#FFCDC9"
  html.text.one: "This is a NEW configmap value (1st)"
  html.text.two: "Another NEW configmap value (2nd)"
  html.text.three: "Don't forget the (3rd) NEW configmap value"  
```


And this is the plain secret - devops-reloader-secret-plain.yml. 

devops-reloader-secret-plain.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-blue-secret
  namespace: devops
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
data:
  html.text.four: SSBhbSBhIHNlY3JldA==
  html.text.five: SSBhbSBhbm90aGVyIHNlY3JldA==
  html.text.six: V2hvPyBNZT8gSSBhbSB0aGUgdGhpcmQgc2VjcmV0
  # html.text.four: VGhpcyBpcyBhIE5FVyBzZWNyZXQ=
  # html.text.five: QW5vdGhlciBORVcgc2VjcmV0
  # html.text.six: QW5kIG9uZSBtb3JlIE5FVyBzZWNyZXQ=  
```

To work with the reloader, we only need to add an annotation. There are several possible configurations, but let's take this one, where the reloader automatically reloads the pod when the configmap or secret used by the pod changes. The application will take some data from a configmap. As well as secret.

devops-reloader-deployment.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name:  devops
  
---

apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: devops
  name: devops-reloader-deployment-blue
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: devops-reloader-blue
  template:
    metadata:
      labels:
        app.kubernetes.io/name: devops-reloader-blue
      annotations:
        # For other configuration, see https://github.com/stakater/Reloader
        reloader.stakater.com/auto: "true" # reloads the pod if configmap or secret used by the pod changes
    spec:
      containers:
      - name: devops-blue
        image: timpamungkas/devops-blue:2.0.0
        resources:
          requests:
            cpu: "0.1"
            memory: 128Mi
          limits:
            cpu: "0.3"
            memory: 384Mi
        ports:
        - name:  http
          containerPort: 8111
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /devops/blue/actuator/health/readiness
            port: 8111
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 5
        livenessProbe:
          httpGet:
            path: /devops/blue/actuator/health/liveness
            port: 8111
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 5
        env:
          - name: DEVOPS_BLUE_HTML_BG_COLOR
            valueFrom:
              configMapKeyRef:
                name: devops-blue-configmap
                key: html.color.background
          - name: DEVOPS_BLUE_HTML_TEXT_COLOR
            valueFrom:
              configMapKeyRef:
                name: devops-blue-configmap
                key: html.color.text
          - name: DEVOPS_BLUE_HTML_TEXT_ONE
            valueFrom:
              configMapKeyRef:
                name: devops-blue-configmap
                key: html.text.one
          - name: DEVOPS_BLUE_HTML_TEXT_TWO
            valueFrom:
              configMapKeyRef:
                name: devops-blue-configmap
                key: html.text.two
          - name: DEVOPS_BLUE_HTML_TEXT_THREE
            valueFrom:
              configMapKeyRef:
                name: devops-blue-configmap
                key: html.text.three
          - name: DEVOPS_BLUE_HTML_TEXT_FOUR
            valueFrom:
              secretKeyRef:                           # secret
                name: devops-blue-secret
                key: html.text.four
          - name: DEVOPS_BLUE_HTML_TEXT_FIVE
            valueFrom:
              secretKeyRef:                           # secret
                name: devops-blue-secret
                key: html.text.five
          - name: DEVOPS_BLUE_HTML_TEXT_SIX
            valueFrom:
              secretKeyRef:                           # secret
                name: devops-blue-secret
                key: html.text.six
  replicas: 1

---

apiVersion: v1
kind: Service
metadata:
  namespace: devops
  name: devops-blue-clusterip
  labels:
    app.kubernetes.io/name: devops-reloader-blue
spec:
  selector:
    app.kubernetes.io/name: devops-reloader-blue
  ports:
  - port: 8111
    name: http

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: devops
  name: ingress-devops-reloader-haproxy-blue
  labels:
    app.kubernetes.io/name: ingress-devops-reloader-haproxy-blue
spec:
  ingressClassName: haproxy
  rules:
  - http:
      paths:
      - path: /devops/blue
        pathType: Prefix
        backend:
          service:
            name: devops-blue-clusterip
            port:
              number: 8111
```

Extract Cluster certificate

    reloader CMD --> kubeseal --controller-name=sealed-secret-controller --controller-namespace=kube-system --fetch-cert > mycert.pem

    # result:

    mycert.pem

```yaml
-----BEGIN CERTIFICATE-----
MIIEzTCCArWgAwIBAgIRALyf7dUF5prXJFN4GxOtkZMwDQYJKoZIhvcNAQELBQAw
...
...
qtJ3yvehS4QWv2LCKgXL4XaTwXZpIUyq+uLx5As0kKWZ
-----END CERTIFICATE-----
```


Seal the secret

    reloader CMD --> kubeseal --cert mycert.pem -o yaml < devops-reloader-secret-plain.yml > devops-reloader-secret-sealed.yml

    # result:

    devops-reloader-secret-sealed.yml

Then apply the sealed secret, configmap, and the deployment.

devops-reloader-secret-sealed.yml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  name: devops-blue-secret
  namespace: devops
spec:
  encryptedData:
    html.text.five: AgA...iwO/
    html.text.four: AgC...JMQ8
    html.text.six: AgA...VzReE=
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      name: devops-blue-secret
      namespace: devops
```

Apply devops-reloader-secret-sealed.yml

    reloader CMD --> kubectl apply -f devops-reloader-secret-sealed.yml

    # result: sealedsecret.bitnami.com/devops-blue-secret created


devops-reloader-configmap.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name:  devops

---

kind: ConfigMap 
apiVersion: v1 
metadata:
  namespace: devops
  name: devops-blue-configmap
immutable: false
data:
  html.color.background: "#F7E396"
  html.color.text: "#5C3E94"
  html.text.one: "Hello there, I am the first configmap"
  html.text.two: "Just another configmap (2nd)"
  html.text.three: "This is the THIRD text from k8s configmap"
  # html.color.background: "#0F828C"
  # html.color.text: "#FFCDC9"
  # html.text.one: "This is a NEW configmap value (1st)"
  # html.text.two: "Another NEW configmap value (2nd)"
  # html.text.three: "Don't forget the (3rd) NEW configmap value"  
```

Apply devops-reloader-configmap.yml

    reloader CMD --> kubectl apply -f devops-reloader-configmap.yml

    # result:
    namespace/devops configured
    configmap/devops-blue-configmap created

Apply devops-reloader-deployment.yml

    CMD --> kubectl apply -f devops-reloader-deployment.yml

    # result:
    namespace/devops unchanged
    deployment.apps/devops-reloader-deployment-blue created
    service/devops-blue-clusterip created
    ingress.networking.k8s.io/ingress-devops-reloader-haproxy-blue created


Make sure minikube tunnel is up

    bash --> minikube tunnel

Check the data and note the text from the configmap and secret.

<img src="pics/postman-reloader-first-configmap.png" width="1200" />
<br>
<br>

Now, go back to the configmap, and change some text.

reloader\devops-reloader-configmap.yml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name:  devops

---

kind: ConfigMap 
apiVersion: v1 
metadata:
  namespace: devops
  name: devops-blue-configmap
immutable: false
data:
  # html.color.background: "#F7E396"
  # html.color.text: "#5C3E94"
  # html.text.one: "Hello there, I am the first configmap"
  # html.text.two: "Just another configmap (2nd)"
  # html.text.three: "This is the THIRD text from k8s configmap"
  html.color.background: "#0F828C"
  html.color.text: "#FFCDC9"
  html.text.one: "This is a NEW configmap value (1st)"
  html.text.two: "Another NEW configmap value (2nd)"
  html.text.three: "Don't forget the (3rd) NEW configmap value"  
```

Reapply the configmap.

    reloader CMD --> kubectl apply -f devops-reloader-configmap.yml

    # result:
    namespace/devops unchanged
    configmap/devops-blue-configmap configured

At this point, see that the reloader is creating a new pod. Wait a while until the new pod becomes ready.

    DMC --> kubectl get pods -n devops

    # result:
    NAME                                                  READY   STATUS    RESTARTS   AGE
    devops-reloader-deployment-blue-685c57f59b-shhdq      1/1     Running   0          11m
    devops-reloader-deployment-blue-78688ffd9d-vmjhs      0/1     Running   0          56s

Make sure minikube tunnel is up

    bash --> minikube tunnel

Make request with Postman and if welook at the HTML again, the text from the configmap will be changed. 

<img src="pics/postman-reloader-second-configmap.png" width="1200" />
<br>
<br>

In the same way, if we change some text on a secret document.

devops-reloader-secret-plain.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devops-blue-secret
  namespace: devops
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
data:
  # html.text.four: SSBhbSBhIHNlY3JldA==
  # html.text.five: SSBhbSBhbm90aGVyIHNlY3JldA==
  # html.text.six: V2hvPyBNZT8gSSBhbSB0aGUgdGhpcmQgc2VjcmV0
  html.text.four: VGhpcyBpcyBhIE5FVyBzZWNyZXQ=
  html.text.five: QW5vdGhlciBORVcgc2VjcmV0
  html.text.six: QW5kIG9uZSBtb3JlIE5FVyBzZWNyZXQ=  
```

Seal it:

    reloader CMD --> kubeseal --cert mycert.pem -o yaml < devops-reloader-secret-plain.yml > devops-reloader-secret-sealed-new.yml

    # result:

    devops-reloader-secret-sealed-new.yml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  name: devops-blue-secret
  namespace: devops
spec:
  encryptedData:
    html.text.five: AgAD...GcE=
    html.text.four: AgCg...==
    html.text.six: AgA...5uQ==
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      name: devops-blue-secret
      namespace: devops
```

Reapply the sealed secret.

    reloader DMC --> kubectl apply -f devops-reloader-secret-sealed-new.yml

    # result: sealedsecret.bitnami.com/devops-blue-secret configured

At this point, see that the reloader is creating a new pod. Wait a while until the new pod becomes ready.

    CMD --> kubectl get pods -n devops

    # result:
    NAME                                                  READY   STATUS    RESTARTS   AGE
  devops-reloader-deployment-blue-5ccb77789-c5xm8       0/1     Running   0          24s
  devops-reloader-deployment-blue-78688ffd9d-vmjhs      1/1     Running   0          10m


Make sure minikube tunnel is up

    bash --> minikube tunnel

Make request with Postman and if we look at the HTML again, the text from secret will be changed.

<img src="pics/postman-reloader-new-secret-access.png" width="1200" />
<br>
<br>




[⬆ Back to top](#top)

