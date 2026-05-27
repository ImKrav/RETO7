# Scalable Jenkins on Kubernetes with Kaniko

A complete, declarative, and step-by-step guide to setting up a highly scalable Jenkins environment on Kubernetes, leveraging **Jenkins Configuration as Code (JCasC)** and **Kaniko** for secure, rootless Docker image builds inside Kubernetes pods.

---

## Architecture Overview

Here is a high-level representation of our scalable setup:

* **Figure 1**: Final Setup (Jenkins Controller and dynamic Jenkins Agents running as Kubernetes Pods).
* **Figure 2**: CI Workflow (Developer commits -> Webhook triggers Jenkins job -> Jenkins agent is dynamically spawned -> Kaniko builds and pushes the Docker image to the registry).
* **Figure 3**: Files Distribution (Folder structure of manifests and Dockerfiles).

---

## Table of Contents
1. [Creating the Jenkins Configuration as Code (JCasC) File](#1-creating-the-jenkins-configuration-as-code-jcasc-file)
   - [Configuring Jenkins Global Security](#configuring-jenkins-global-security)
   - [Connecting Jenkins with the Cluster](#connecting-jenkins-with-the-cluster)
2. [Installing Jenkins](#2-installing-jenkins)
   - [Jenkins Controller Image](#jenkins-controller-image)
   - [Jenkins Agent Image with Kaniko](#jenkins-agent-image-with-kaniko)
3. [Deploying Everything to Kubernetes](#3-deploying-everything-to-kubernetes)
   - [Service Account & RBAC permissions](#service-account--rbac-permissions)
   - [Persistent Storage (PVC)](#persistent-storage-pvc)
   - [Docker Registry Credentials Secret](#docker-registry-credentials-secret)
   - [Jenkins Deployment](#jenkins-deployment)
   - [Jenkins Service](#jenkins-service)
4. [Running Your First Jenkins Job](#4-running-your-first-jenkins-job)
   - [The `build.sh` Script](#the-buildsh-script)
   - [Creating the Job in Jenkins](#creating-the-job-in-jenkins)

---

## 1. Creating the Jenkins Configuration as Code (JCasC) File

We will use the **Jenkins Configuration as Code (JCasC)** plugin to manage our Jenkins setup. JCasC provides a declarative, GitOps-friendly way of managing Jenkins configuration using a single YAML file. 

Create a file named `casc.yaml`. This file defines global security, credentials, clouds, and agent pod templates.

### The Complete `casc.yaml` File

```yaml
jenkins:
  systemMessage: "Welcome to Jenkins!"
  numExecutors: 0
  remotingSecurity:
    enabled: true
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: ${JENKINS_ADMIN_ID}
          password: ${JENKINS_ADMIN_PASSWORD}
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
  clouds:
    - kubernetes:
        skipTlsVerify: true
        useJenkinsProxy: false
        jenkinsUrl: "http://jenkins-service:8080"
        jenkinsTunnel: "jenkins-service:50000"
        maxRequestsPerHost: 32
        name: "kubernetes"
        readTimeout: 15
        podLabels:
          - key: jenkins
            value: agent
        templates:
          - name: "jenkins-agent"
            label: "jenkins-agent"
            hostNetwork: false
            nodeUsageMode: "NORMAL"
            serviceAccount: "jenkins"
            imagePullSecrets:
              - name: regcred
            yamlMergeStrategy: "override"
            containers:
              - name: jnlp
                image: "<dockerhub-user>/jenkins-jnlp-kaniko"
                alwaysPullImage: true
                workingDir: "/home/jenkins/agent"
                command: ""
                args: ""
                livenessProbe:
                  failureThreshold: 1
                  initialDelaySeconds: 2
                  periodSeconds: 3
                  successThreshold: 4
                  timeoutSeconds: 5
            volumes:
              - secretVolume:
                  mountPath: /kaniko/.docker
                  secretName: regcred
unclassified:
  location:
    url: http://127.0.0.1:8080/
```

---

### Configuring Jenkins Global Security

By default, a fresh Jenkins instance is unsecured. To protect our instance, we configure JCasC as follows:

```yaml
jenkins:
  remotingSecurity:
    enabled: true
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: ${JENKINS_ADMIN_ID}
          password: ${JENKINS_ADMIN_PASSWORD}
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
```

* **`remotingSecurity`**: Secures communication between the Jenkins controller and the dynamically spawned agent pods.
* **`securityRealm`**: Configured to `local` database authentication.
  - **`allowsSignup: false`**: Disables the signup form on the Jenkins login page.
  - **`users`**: Creates a single administrator account using credentials defined in the environment variables `${JENKINS_ADMIN_ID}` and `${JENKINS_ADMIN_PASSWORD}`.
* **`authorizationStrategy`**: Configured to use a Matrix Authorization strategy, assigning administrator rights (`Overall/Administer`) to `admin`, and read access (`Overall/Read`) to authenticated users.

Additionally, to ensure all workloads are built on dynamically spawned agents and **not** on the controller itself, we set the executors count on the controller to zero:

```yaml
jenkins:
  systemMessage: "Welcome to Jenkins!"
  numExecutors: 0
```

---

### Connecting Jenkins with the Cluster

To spin up builds dynamically as Kubernetes Pods, we configure the Kubernetes Cloud Provider inside JCasC.

```yaml
jenkins:
  clouds:
    - kubernetes:
        serverUrl: "https://<kubernetes_control_plane_ip>"
        jenkinsUrl: "http://jenkins-service:8080"
        jenkinsTunnel: "jenkins-service:50000"
        skipTlsVerify: false
        useJenkinsProxy: false
        maxRequestsPerHost: 32
        name: "kubernetes"
        readTimeout: 15
        podLabels:
          - key: jenkins
            value: agent
```

> [!NOTE]
> * **`serverUrl`**: Points to your Kubernetes Control Plane IP. This address can be omitted if you are running inside Minikube.
> * **`jenkinsUrl`**: Points to the service exposing the controller: `http://jenkins-service:8080`.
> * **`jenkinsTunnel`**: The service endpoint and port (`50000`) that Jenkins agents use to communicate with the controller over JNLP protocol.
> * **`podLabels`**: Applies tags (`key=jenkins`, `value=agent`) to all dynamically created agent pods.

#### Agent Pod Template Configuration

Each build pod template contains configuration detailing the pod specification:

```yaml
        templates:
          - name: "jenkins-agent"
            label: "jenkins-agent"
            hostNetwork: false
            nodeUsageMode: "NORMAL"
            serviceAccount: "jenkins"
            imagePullSecrets:
              - name: regcred
            yamlMergeStrategy: "override"
```

* **`serviceAccount`**: Set to `jenkins` so agents inherit correct RBAC permissions to work with the Kubernetes API.
* **`imagePullSecrets`**: Provides `regcred` secret credentials to download the agent Docker image from private registries.

#### Container Template Configuration

Within the Pod template, we configure the container containing our build agent:

```yaml
            containers:
              - name: jnlp
                image: "<your_dockerhub_user>/jenkins-jnlp-kaniko"
                workingDir: "/home/jenkins/agent"
                command: ""
                args: ""
                livenessProbe:
                  failureThreshold: 1
                  initialDelaySeconds: 2
                  periodSeconds: 3
                  successThreshold: 4
                  timeoutSeconds: 5
            volumes:
              - secretVolume:
                  mountPath: /kaniko/.docker
                  secretName: regcred
```

* **`image`**: Custom agent Docker image that bundles the Jenkins inbound-agent with Kaniko executor.
* **`volumes`**: We mount our `regcred` Kubernetes secret to `/kaniko/.docker`. This contains our Docker registry configuration that Kaniko will use to securely push newly built images to the registry.

---

## 2. Installing Jenkins

We will customize the official Jenkins base image to package all our required plugins and apply JCasC on startup.

### Jenkins Controller Image

Create a `Dockerfile` for the Jenkins controller:

```dockerfile
FROM jenkins/jenkins
ENV CASC_JENKINS_CONFIG /usr/local/casc.yaml
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
COPY casc.yaml /usr/local/casc.yaml
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
```

* **`CASC_JENKINS_CONFIG`**: Environment variable pointing to our declarative configuration file `casc.yaml`.
* **`JAVA_OPTS`**: Disables the first-time setup wizard (`runSetupWizard=false`).
* **`jenkins-plugin-cli`**: Automatically downloads and installs the plugins listed inside `plugins.txt`.

Create the companion `plugins.txt` file listing required plugins:

```text
configuration-as-code
matrix-auth
ssh-slaves
email-ext
mailer
slack
htmlpublisher
greenballs
simple-theme-plugin
kubernetes
git
github
```

Build, tag, and push the Jenkins controller image to your Docker Hub:

```bash
# Build the image
docker build -t <your_dockerhub_user>/jenkins-controller-kaniko .

# Push to registry
docker login
docker push <your_dockerhub_user>/jenkins-controller-kaniko
```

---

### Jenkins Agent Image with Kaniko

The agent pod needs tools to build container images. We use a multi-stage Dockerfile to combine the official **Jenkins inbound-agent** and the **Kaniko executor**.

Create a new `Dockerfile` for the Jenkins agent:

```dockerfile
FROM gcr.io/kaniko-project/executor:v1.13.0 as kaniko
FROM jenkins/inbound-agent
COPY --from=kaniko /kaniko /kaniko
WORKDIR /kaniko
USER root
```

Build and push the agent image to your Docker Hub:

```bash
# Build the agent image
docker build -t <your_dockerhub_user>/jenkins-jnlp-kaniko .

# Push to registry
docker push <your_dockerhub_user>/jenkins-jnlp-kaniko
```

---

## 3. Deploying Everything to Kubernetes

Before starting the deployment, prepare your Kubernetes environment (Minikube in this example).

```bash
# Clean up any existing state
minikube delete

# Start a fresh Minikube cluster using Docker driver
minikube start --driver=docker
```

---

### Service Account & RBAC Permissions

Create a `jenkins-sa-crb.yaml` file to define a service account and bind it to the `cluster-admin` role. This allows the Jenkins pod to talk to the Kubernetes API and deploy pods.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: jenkins
  name: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crb
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: default
```

Apply the manifest:

```bash
kubectl apply -f jenkins-sa-crb.yaml
```

---

### Persistent Storage (PVC)

To persist Jenkins data (jobs, configuration, build history) beyond a Pod lifecycle, create a `jenkins-pvc.yaml` manifest:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pv-claim
  labels:
    app: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```

Apply the persistent volume claim:

```bash
kubectl apply -f jenkins-pvc.yaml
```

---

### Docker Registry Credentials Secret

Create a Kubernetes Secret named `regcred` to authenticate both Jenkins agent pulls and Kaniko image uploads:

```bash
kubectl create secret docker-registry regcred \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-server=https://index.docker.io/v1/
```

---

### Jenkins Deployment

Create a `jenkins-deployment.yaml` file. Ensure you replace `<dockerhub-user>` with your username and `<jenkins-admin-pass>` with a secure password of your choice.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      volumes:
        - name: jenkins-pv-storage
          persistentVolumeClaim:
            claimName: jenkins-pv-claim
      containers:
        - name: jenkins
          image: <dockerhub-user>/jenkins-controller-kaniko
          env:
            - name: JENKINS_ADMIN_ID
              value: admin
            - name: JENKINS_ADMIN_PASSWORD
              value: <jenkins-admin-pass>
          ports:
            - containerPort: 8080
            - containerPort: 50000
          volumeMounts:
            - mountPath: "/var/jenkins_home"
              name: jenkins-pv-storage
      imagePullSecrets:
        - name: regcred
      initContainers:
        - name: volume-mount-data-log
          image: busybox
          imagePullPolicy: Always
          command: ["sh", "-c", "chown -R 1000:1000 /var/jenkins_home"]
          volumeMounts:
            - mountPath: "/var/jenkins_home"
              name: jenkins-pv-storage
```

Apply the deployment:

```bash
kubectl apply -f jenkins-deployment.yaml
```

---

### Jenkins Service

Expose Jenkins to external HTTP traffic (port 8080) and internal Agent traffic (port 50000) using a LoadBalancer Service inside `jenkins-svc.yaml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  labels:
    app: jenkins-svc
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      name: app
    - port: 50000
      targetPort: 50000
      protocol: TCP
      name: jnlp
  selector:
    app: jenkins
```

Apply the service:

```bash
kubectl apply -f jenkins-svc.yaml
```

Get the URL of the service to access the UI from your host machine:

```bash
minikube service jenkins-service --url
```

* **Figure 4**: Jenkins login page (verifies JCasC global security is running). Log in using `admin` and the password you defined.
* **Figure 5**: Jenkins Home Page.

---

## 4. Running Your First Jenkins Job

We will create a pipeline to build a microservice using Kaniko from the following repository: [ejemplo-jenkins-kaniko](https://github.com/jilopez-udem/ejemplo-jenkins-kaniko).

### The `build.sh` Script

Our source repository contains a `build.sh` script to trigger Kaniko inside the agent pod. The script accepts two arguments (`IMAGE_ID` and `IMAGE_TAG`), writes Docker configuration containing our secrets into the home directory, and invokes `/kaniko/executor`:

```bash
#!/bin/bash
mkdir -p /kaniko/.dockerconfig && ln -s /kaniko/.docker/.dockerconfigjson /kaniko/.dockerconfig/config.json

IMAGE_ID=$1
IMAGE_TAG=$2
export DOCKER_CONFIG=/kaniko/.dockerconfig

/kaniko/executor \
  --context $(pwd) \
  --dockerfile $(pwd)/Dockerfile \
  --destination $IMAGE_ID:$IMAGE_TAG \
  --force
```

---

### Creating the Job in Jenkins

1. Go to your Jenkins Dashboard and select **New Item**.
2. Select **Freestyle Job**, name your job, and click **Next**.
3. Under **Source Code Management**, select **Git** and add your repository URL. Specify the build branch (e.g., `*/main`).
   * *See **Figure 6**: Jenkins pipeline definition.*
4. Under **Build Steps**, select **Add build step** | **Execute shell**.
5. Input the following shell command to call `build.sh`:
   ```bash
   ./build.sh <your_dockerhub_user>/ejemplo latest
   ```
6. Click **Save**.

Now, click **Build Now**. Jenkins will:
* Dynamically deploy a `jenkins-agent` pod inside Kubernetes.
* Connect to it via JNLP.
* Check out the code, run `build.sh` using Kaniko to compile the Docker image, run tests, and securely push the final image to Docker Hub using the credentials stored in the `regcred` secret.

* **Figure 7**: Jenkins - Execute Shell configuration.
* **Figure 8**: Jenkins Job Page.
* **Figure 9**: Jenkins Console Output (showing successful test execution and successful image push).
