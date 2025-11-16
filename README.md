# Complete CI/CD DevOps Project 🚀
### Deploy Python Flask App on Kubernetes cluster with GitOps Approach. 

![alt text](imgs/arch.png)

---
### Workflow:
Whenever Developer writing/changes a code and push into master/main branch, GitHub Pipeline will triggered and it will test the code with Flake8 and containerized the application with new tag and push into artifacts(dockerhub) and also GitHub Actions pipeline will update Kubernetes Manifests file with new image tag then ArgoCD will look for new changes in Manifests file and will rollout new application in kubernetes. 
```
│   app.py
│   LICENSE
│   README.md
│   requirements.txt
│
├───deploy
│       deploy.yaml
│       svc.yaml
│
├───static
│       style.css
│
└───templates
        index.html
```
---
#### What you will learn:
- Git for version control
- VS Code Editor
- Docker for testing locally
- Minikube for Kubernetes 1 Node Arch. 
- GitHub for storing code
- GitHub Actions for Continous Integrity Pipeline 
- ArgoCD for Continous Deployment Pipeline
- Python Application
    - Flask Framework
    - Flake8 Module for Linting testing  
---
## Test Application Locally. 
Whenever we are creating pipeline, it is best practice to test application locally.
- Application prequisities. 
  - Python 3.9 
  - pip installed

- Clone/Fork the Repo. 
    ```
    git clone https://github.com/Dinesh-devops-123/Flask-App-Github-Actions-AgroCD.git
    cd Flask-App-GitHub-Actions-ArgoCD
    ```
- Install Dependence
    ```
    pip install -r requirements.txt
    ```
- Run locally. 
    ```
    python app.py
    ```
- Access the application.
    ```
    http://localhost:5000
    ```

Note: This application is running on 5000 port, but if you want to change, you can change the port in app.py script.

---
Write Dockerfile
```
# Step 1: Base image
FROM python:3.9-slim

# Step 2: Set working directory
WORKDIR /app

# Step 3: Copy application code to the container
COPY . .

# Step 4: Install dependencies
RUN pip install -r requirements.txt

# Step 5: Expose the application port
EXPOSE 5000

# Step 6: Define the command to run the application
CMD ["python", "app.py"]
```
#### Lets Build and Run the Container
1. Build the Image: Run the following in the directory containing your Dockerfile:
```
docker build -t dineshops/demo-app:v1 .
```
Note: you need to change the name of your image, according to your dockerhub username.

2. Lets create container with image.
 ```
 docker run -d -p hostport:containerport --name=container_name img_name:v1
 docker run -d -p 5000:5000 --name=demo-app demo-app
 ```

 If everything is working fine and you are able to access application with https://localhost:5000 then next step is to write a GitHub Pipeline.

## CI Pipeline with GitHub Actions
1. Create a directory inside your project.
    ```
    mkdir -p .github/Workflows
    ```
2. Create your first pipeline for TEST and BUILD the image. make sure it should be yaml file
    ```
    name: Test and Build

    on:
    push:
        branches:
        - master
        paths:
        - '**/*'

    jobs:
    build:
        runs-on: ubuntu-latest

        steps:
        #Setting up environment
        - name: Checkout code
            uses: actions/checkout@v2

        - name: Setup Python
            uses: actions/setup-python@v2
            with:
            python-version: '3.9'

        - name: Docker Setup
            uses: docker/setup-buildx-action@v2

        - name: Install dependencies
            run: |
            python -m pip install --upgrade pip
            pip install -r requirements.txt
            pip install flake8
            
        # Test the Code
        - name: Run Linting tests
            run: |
            flake8 --ignore=E501,F401 .
        
        - name: Docker Credentials
            uses: docker/login-action@v2
            with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}
        
        - name: Docker tag
            id: version
            run: |
            VERSION=v$(date +"%Y%m%d%H%M%S")
            echo "VERSION=$VERSION" >> $GITHUB_ENV

        # Build the Docker Image
        - name: Build Docker Image
            run: |
            docker build . -t dineshops/demo-app:${{ env.VERSION }} 
        
        # Push the Docker Image
        - name: Push Docker Image
            run: |
            docker push dineshops/demo-app:${{ env.VERSION }}
        
        # UPdate the K8s Manifest Files
        - name: Update K8s Manifests
            run: |
            cat deploy/deploy.yaml
            sed -i "s|image: dineshops/demo-app:.*|image: dineshops/demo-app:${{ env.VERSION }}|g" deploy/deploy.yaml
            cat deploy/deploy.yaml

        # Update Github
        - name: Commit the changes
            run: |
            git config --global user.email "<dineshmoorthi8300@gmail.com>"
            git config --global user.name "GitHub Actions Bot"
            git add deploy/deploy.yaml
            git commit -m "Update deploy.yaml with new image version - ${{ env.VERSION }}"
            git remote set-url origin https://github-actions:${{ secrets.GITHUB_TOKEN }}@github.com/dinesh-devops-123/Flask-App-GitHub-Actions-ArgoCD.git
            git push origin master
    ```

1. Make sure setup your docker Personal Access token into github repo. 

## Setup ArgoCD in Minikube

Note: You can setup Argo CD in any cluster, instructions are same. 

- First install Minikube:
    Installation guide for installing Minikube. 
    [Minikube.sigs.k8s.io](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)

---
- Install Argo CD
    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
    ```

- Verify if ArgoCD is running:
    ```
    kubectl get all -n argocd
    ```
    Output
    ```    
            NAME                                                    READY   STATUS              RESTARTS   AGE
    pod/argocd-application-controller-0                     0/1     ContainerCreating   0          55s
    pod/argocd-applicationset-controller-7b6ff755dc-sl2nd   0/1     ContainerCreating   0          55s
    pod/argocd-dex-server-584f7d88dc-sfk4v                  0/1     Init:0/1            0          55s
    pod/argocd-notifications-controller-67cdd486c6-zznqn    0/1     ContainerCreating   0          55s
    pod/argocd-redis-6dbb9f6cf4-9s5fx                       0/1     Init:0/1            0          55s
    pod/argocd-repo-server-57bdcb5898-rklsv                 0/1     Init:0/1            0          55s
    pod/argocd-server-57d9cc9bcf-slmrd                      0/1     ContainerCreating   0          55s

    NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    service/argocd-applicationset-controller          ClusterIP   10.100.121.169   <none>        7000/TCP,8080/TCP            52s
    service/argocd-dex-server                         ClusterIP   10.109.212.68    <none>        5556/TCP,5557/TCP,5558/TCP   56s
    service/argocd-metrics                            ClusterIP   10.96.144.188    <none>        8082/TCP                     56s
    service/argocd-notifications-controller-metrics   ClusterIP   10.96.81.128     <none>        9001/TCP                     56s
    service/argocd-redis                              ClusterIP   10.104.154.194   <none>        6379/TCP                     56s
    service/argocd-repo-server                        ClusterIP   10.99.25.108     <none>        8081/TCP,8084/TCP            56s
    service/argocd-server                             ClusterIP   10.102.201.60    <none>        80/TCP,443/TCP               56s
    service/argocd-server-metrics                     ClusterIP   10.104.218.14    <none>        8083/TCP                     56s

    NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/argocd-applicationset-controller   0/1     1            0           56s
    deployment.apps/argocd-dex-server                  0/1     1            0           55s
    deployment.apps/argocd-notifications-controller    0/1     1            0           55s
    deployment.apps/argocd-redis                       0/1     1            0           55s
    deployment.apps/argocd-repo-server                 0/1     1            0           55s
    deployment.apps/argocd-server                      0/1     1            0           55s

    NAME                                                          DESIRED   CURRENT   READY   AGE
    replicaset.apps/argocd-applicationset-controller-7b6ff755dc   1         1         0       55s
    replicaset.apps/argocd-dex-server-584f7d88dc                  1         1         0       55s
    replicaset.apps/argocd-notifications-controller-67cdd486c6    1         1         0       55s
    replicaset.apps/argocd-redis-6dbb9f6cf4                       1         1         0       55s
    replicaset.apps/argocd-repo-server-57bdcb5898                 1         1         0       55s
    replicaset.apps/argocd-server-57d9cc9bcf                      1         1         0       55s

    NAME                                             READY   AGE
    statefulset.apps/argocd-application-controller   0/1     55s
    ```

---
- Access ArgoCD With configuring NodePort 
    ```
    kubectl patch service argocd-server -n argocd --type=merge -p '{\"spec\": {\"type\": \"NodePort\"}}'
    minikube ip
    ```
- Verify if ArgoCD server running as NodePort.
   ```
   kubectl get svc -n argocd
   ``` 
   Output
   ```
    NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
    argocd-applicationset-controller          ClusterIP   10.100.121.169   <none>        7000/TCP,8080/TCP            9m37s
    argocd-dex-server                         ClusterIP   10.109.212.68    <none>        5556/TCP,5557/TCP,5558/TCP   9m41s
    argocd-metrics                            ClusterIP   10.96.144.188    <none>        8082/TCP                     9m41s
    argocd-notifications-controller-metrics   ClusterIP   10.96.81.128     <none>        9001/TCP                     9m41s
    argocd-redis                              ClusterIP   10.104.154.194   <none>        6379/TCP                     9m41s
    argocd-repo-server                        ClusterIP   10.99.25.108     <none>        8081/TCP,8084/TCP            9m41s
    argocd-server                             NodePort    10.102.201.60    <none>        80:31209/TCP,443:32389/TCP   9m41s
    argocd-server-metrics                     ClusterIP   10.104.218.14    <none>        8083/TCP                     9m41s
   ```
- Grab ArgoCD secret for accessing UI
   ```
   kubectl get secrets -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

   # passwd 
   kubectl get secret argocd-initial-admin-secret -n argocd `
  -o jsonpath="{.data.password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

   ```

- Start Minkube Service. 
   ```
    minikube service argocd-server -n argocd
    ```
    Output
    
    ┌───────────┬───────────────┬─────────────┬───────────────────────────┐
    │ NAMESPACE │     NAME      │ TARGET PORT │            URL            │
    ├───────────┼───────────────┼─────────────┼───────────────────────────┤
    │ argocd    │ argocd-server │ http/80     │ http://192.168.49.2:31209 │
    │           │               │ https/443   │ http://192.168.49.2:32389 │
    └───────────┴───────────────┴─────────────┴───────────────────────────┘
    * Starting tunnel for service argocd-server./
    ┌───────────┬───────────────┬─────────────┬────────────────────────┐
    │ NAMESPACE │     NAME      │ TARGET PORT │          URL           │
    ├───────────┼───────────────┼─────────────┼────────────────────────┤
    │ argocd    │ argocd-server │             │ http://127.0.0.1:62939 │
    │           │               │             │ http://127.0.0.1:62940 │
    └───────────┴───────────────┴─────────────┴────────────────────────┘
    * Starting tunnel for service argocd-server.
    [argocd argocd-server  http://127.0.0.1:62939
    http://127.0.0.1:62940]

   ```
   Username: admin
   password: secret(please check above command)

   ![alt text](imgs/ui.png)

---
Setup our Continous deployment. 

- Select New App.
![alt text](imgs/setting1.png)
![alt text](imgs/setting2.png)
---
- Syncing your manifests files:
![alt text](imgs/sync.png)
---

- Successfully Deployed our app:
![alt text](imgs/deployed.png)
---
Access Application with below command.
```
minikube service list
```
Output
```
|-------------|-----------------------------------------|--------------|-----------------------------|
|  NAMESPACE  |                  NAME                   | TARGET PORT  |             URL             |
|-------------|-----------------------------------------|--------------|-----------------------------|
| argocd      | argocd-applicationset-controller        | No node port |                             |
| argocd      | argocd-dex-server                       | No node port |                             |
| argocd      | argocd-metrics                          | No node port |                             |
| argocd      | argocd-notifications-controller-metrics | No node port |                             |
| argocd      | argocd-redis                            | No node port |                             |
| argocd      | argocd-repo-server                      | No node port |                             |
| argocd      | argocd-server                           | http/80      | http://172.29.213.129:30692 |
|             |                                         | https/443    | http://172.29.213.129:31365 |
| argocd      | argocd-server-metrics                   | No node port |                             |
| default     | kubernetes                              | No node port |                             |
| default     | weather-check-service                   |         5000 | http://172.29.213.129:30008 |
| kube-system | kube-dns                                | No node port |                             |
|-------------|-----------------------------------------|--------------|-----------------------------|
```
---
Application running on http://172.29.213.129:30008

![alt text](imgs/application.png)