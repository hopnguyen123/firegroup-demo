# 0. Tổng quát
## 0.1 Sơ đồ triển khai dự án
- ![alt text](45.png)
- Chúng ta sẽ triển khai từ đầu các bước bao gồm:
    + Setup Gitlab repo
    + Setup Jenkins on EC2 instance
    + Setup Dockerhub
    + Setup K8S cluster via minikube (on EC2 instance)
    + Setup connection Jenkins and Gitlab
    + Setup connection Jenkins and Dockerhub
    + Setup connection Jenkins and K8S cluster
    + Create pipeline
        - Clone repo
        - Build docker image
        - Login to dockerhub
        - Push image to Dockerhub
        - Deploy image to 3 pod in K8S Cluster
    + Tigger Pipeline khi có push event từ Gitlab repo
# 1. Tạo Gitlab repo
- Tạo gitlab repo `firegroup-demo`
    + ![alt text](1.png)
- Tiến hành tạo file `README.md` và đẩy lên repo `firegroup-demo`
    + ![alt text](2.png)
    + ![alt text](3.png)
- Tiến hành đẩy toàn bộ source code lên repo `firegroup-demo`
    + ![alt text](4.png)

# 2. Tạo EC2 Instance trên AWS
- Trên EC2 Instance này em sẽ cài đặt:
    + `Jenkins server`: bằng cách chạy `docker compose`
    + `K8S cluster`: cài đặt thông qua `minikube`
## 2.1 Tạo EC2 
- Thông tin cấu hình
    + Name: `jenkins-k8s`
    + OS: `Ubuntu 20.04`
    + Instance type: `t3.medium`
    + Storage: `30GiB gp3`
    + Key pair: `jenkins-k8s-key`
    + ![alt text](5.png)
    + ![alt text](6.png)
    + ![alt text](7.png)
    + ![alt text](8.png)

## 2.2 SSH từ local đến EC2 Instance
- Ta dùng lệnh: `ssh -i jenkins-k8s-key.pem ubuntu@3.93.1.93`
- ![alt text](9.png)
- ![alt text](10.png)

# 3. Cài đặt Docker engine trên EC2 instance
- Ta sẽ cài đặt theo hướng dẫn này [install docker](https://docs.docker.com/engine/install/ubuntu/)
    + `sudo apt-get update`
    + `sudo apt-get install ca-certificates curl`
    + `sudo install -m 0755 -d /etc/apt/keyrings`
    + `sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc`
	+ `sudo chmod a+r /etc/apt/keyrings/docker.asc`
    + ```
        echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
        $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null 
      ```
    + `sudo apt-get update`
    + `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
    + `sudo docker run hello-world`
    + `sudo groupadd docker`
    + `sudo usermod -aG docker $USER`
    + `newgrp docker`
    + `sudo systemctl enable docker.service`
    + `docker run hello-world`
- Kiểm tra lại sau khi cài xong
    + ![alt text](11.png)

# 4. Cài đặt K8S Cluster thông qua minikube
## 4.1 Cài đặt `minikube` 
- Ta sẽ cài đặt theo hướng dẫn này [install minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)
- Ta gõ các lệnh để cài
    + `curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64`
    + `sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64`
    + `minikube start`
    + `minikube status`
- Kiểm tra lại sau khi cài xong `minikube`
    + ![alt text](12.png)
    + ![alt text](13.png)

## 4.2 Cài đặt `K8S cluster`
- Ta sẽ dùng cách lệnh này để cài
    + `sudo apt-get update`
    + `sudo apt-get install -y apt-transport-https ca-certificates curl gnupg`
    + `curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
    + `sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
    + `echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list`
    + `sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list`
    + `sudo apt-get update`
    + `sudo apt-get install -y kubectl`
- Kiểm tra sau khi cài xong `K8S cluster`
    + `kubectl get pods`
    + `kubectl cluster-info`
        - Địa chỉ của cluster: https://192.168.49.2:8443
    + ![alt text](14.png)

# 5. Cài đặt Jenkins thông qua docker compose
## 5.1 Cài đặt Jenkins 
- Tạo `docker-compose.yml` file
    + ``` yaml
        version: '3.8'

        services:
          jenkins:
            image: jenkins/jenkins:lts
            container_name: jenkins-kube
            user: root
            restart: unless-stopped
            ports:
              - "8080:8080"
              - "50000:50000"
            networks:
              - minikube
            environment:
              - DOCKER_GID=${DOCKER_GID}
            volumes:
              - /home/ubuntu/jenkins_home:/var/jenkins_home
              - /var/run/docker.sock:/var/run/docker.sock
              - /usr/bin/docker:/usr/bin/docker
              - /usr/bin/kubectl:/usr/bin/kubectl

        networks:
          minikube:
            external: true

      ```
- Kiểm tra sau khi chạy xong
    + ![alt text](15.png)

## 5.2 Mở inbound rule ở EC2 instance
- Mở inbound rule để truy cập Jenkins UI
    + ![alt text](16.png)

## 5.3 Các thiết lập Jenkins ban đầu
- Cài đặt các plugin ban đầu
    + ![alt text](17.png)
    + ![alt text](18.png)
    + ![alt text](19.png)

# 6. Kết nối giữa Jenkins và Gitlab
## 6.1 Tạo Gitlab access token
- Token name: `jenkins_trigger_token`
    + scopes: `api`
    + token: `glpat-s_63Tqvt52TqKVs7B7_T`
- ![alt text](20.png)

## 6.2 Tạo Gitlab connection từ Jenkins UI
- Tạo credential `jenkins-gitlab-token`
    + ![alt text](21.png)
- Kiểm tra kết nối bằng nút `Test Connection` và đã thành công
    + ![alt text](22.png)

## 6.3 Tạo 1 pipeline mới trên Jenkins
- Thiết lập cấu hình ban đầu cho pipeline
    + ![alt text](23.png)
    + ![alt text](24.png)
    + ![alt text](25.png)
    + Tạo `Credential`
        - ![alt text](26.png)
    + ![alt text](27.png)
        - Branch: `main`
        - Script path: `Jenkinsfile`
## 6.4 Tạo webhook trên Gitlab UI
- Tạo `token` trên `Jenkins UI`
    + ![alt text](28.png)
- Thêm Jenkins token trên Gitlab webhook
    + ![alt text](29.png)
    + URL: `http://admin:118bc3b8dbaf994a94812d0d053b91a089@3.93.1.93:8080/project/firegroup-demo`
    + ![alt text](30.png)
        + Kiểm thành công và nhận được thông báo `Hook executed successfully: HTTP 200`

# 7. Kết nối giữa Jenkins và K8S cluster
- Đọc file `ca.crt`
    + `cat /home/ubuntu/.minikube/ca.crt | base64 -w 0; echo;`
    + ![alt text](31.png)
- Tạo các `object` cần thiết trong `K8S cluster`
    + namespace: `jenkins`
    + sa: `jenkins`
    + clusterrole: `cluster-admin`
    + clusterrolebinding: `jenkins-binding`
    + ![alt text](32.png)
- Cấu hình cloud tại: `http://3.93.1.93:8080/manage/cloud/`
    + ![alt text](33.png)
        - Kubernetes server certificate keys: nội dung của `/home/ubuntu/.minikube/ca.crt`
    + ![alt text](34.png)
        + Tạo credentials: sa-k8s-token
        - Secret: nội dung của token `jenkins`
    + ![alt text](35.png)
        - Kiểm tra kết nối thành công bằng cách bấm nút `Test connection`

# 8. Kết nối giữa Jenkins và Dockerhub
- Tạo kết nối này để push docker image lên dockerhub
## 8.1 Cài đặt docker tool tại Jenkins UI
- ![alt text](36.png)
## 8.2 Tại dockerhub
- Tạo Access token
    + ![alt text](37.png)
- Tạo repo để lưu image
    + ![alt text](39.png)
## 8.3 Tạo Credential tại Jenkins UI
- ID: jenkins-dockerhub
    + ![alt text](38.png)

# 9. Tiến hành tạo pipeline
- Sau khi kết nối `Jenkins` với `Gitlab`, `Dockerhub`, `K8S cluster` hoàn thành thì ta sẽ tiến hành đi viết `Jenkinsfile`
## 9.0 Mount thêm các file cần thiết cho Jenkins 
- Chúng ta mount các file cần thiết để Jenkins có thể dùng kubectl để giao tiếp với K8S cluster
- `docker compose down`
- `vi docker-compose.yml`
- ``` yaml
    version: '3.8'

    services:
    jenkins:
        image: jenkins/jenkins:lts
        container_name: jenkins-kube
        user: root
        restart: unless-stopped
        ports:
        - "8080:8080"
        - "50000:50000"
        networks:
        - minikube
        environment:
        - DOCKER_GID=${DOCKER_GID}
        volumes:
        - /home/ubuntu/jenkins_home:/var/jenkins_home
        - /var/run/docker.sock:/var/run/docker.sock
        - /usr/bin/docker:/usr/bin/docker
        - /usr/bin/kubectl:/usr/bin/kubectl
        - ~/.kube/config:/var/jenkins_home/.kube/config
        - ~/.minikube:/var/jenkins_home/.minikube

    networks:
    minikube:
        external: true
  ```
- `docker compose up -d`

## 9.1 Debug tại dockerfile
- Tại `docker/Dockerfile` thiếu copy thư mục `lib` vào `/var/www/lib` để chạy `composer` bên dưới
    + ![alt text](40.png)
## 9.2 Viết Jenkinsfile
- ``` groovy
        pipeline {
            agent any

            environment {
                DOCKER_HUB_CREDENTIALS_ID = "jenkins-dockerhub"
                DOCKER_HUB_REPO = "hopnguyenn/firegroup-demo"
                DOCKER_IMAGE_TAG = ""
                K8S_NAMESPACE = "jenkins"
            }

            stages {
                stage('Checkout') {
                    steps {
                        script {
                            checkout scm
                            DOCKER_IMAGE_TAG = sh(script: "date +'%Y%m%d-%H%M'", returnStdout: true).trim()
                        }
                    }
                }

                stage('Build Docker Image') {
                    steps {
                        script {
                            sh """
                                ls -ltr
                                docker build -t ${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG} -f docker/Dockerfile .
                                docker images
                            """
                        }
                    }
                }

                stage('Login to Docker Hub') {
                    steps {
                        script {
                            withCredentials([usernamePassword(credentialsId: DOCKER_HUB_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                                sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USER --password-stdin"
                            }
                        }
                    }
                }

                stage('Tag and Push') {
                    steps {
                        script {
                            sh """
                                docker tag ${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG} ${DOCKER_HUB_REPO}:latest
                                docker push ${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG}
                                docker push ${DOCKER_HUB_REPO}:latest
                            """
                        }
                    }
                }

                stage ('Deploy pod'){
                    steps {
                        script {
                            def deploymentYaml = """
                            apiVersion: apps/v1
                            kind: Deployment
                            metadata:
                            name: my-app
                            namespace: jenkins
                            spec:
                            replicas: 3
                            selector:
                                matchLabels:
                                    app: my-app
                            template:
                                metadata:
                                labels:
                                    app: my-app
                                spec:
                                containers:
                                - name: my-app
                                    image: hopnguyenn/firegroup-demo:${DOCKER_IMAGE_TAG}
                                    ports:
                                    - containerPort: 80
                            """         
                            writeFile file: 'deployment.yml', text: deploymentYaml

                            sh '''
                                cp /var/jenkins_home/.kube/config /var/jenkins_home/.kube/config.tmp
                                sed -i 's|/home/ubuntu/.minikube/|/var/jenkins_home/.minikube/|g' /var/jenkins_home/.kube/config.tmp
                                export KUBECONFIG=/var/jenkins_home/.kube/config.tmp
                                
                                kubectl apply -f deployment.yml --namespace=jenkins 
                            '''
                        }             
                    }
                }
            }
        }
  ```
- `Jenkinsfile` sẽ có các phần sau
    + `environment`: định nghĩa các biến môi trường cho pipeline
    + `stages`: định nghĩa các stage của pipeline
        - `Checkout`: clone repo
        - `Build Docker Image`: Tạo docker image từ docker file
        - `Login to Docker Hub`: Đăng nhập vào dockerhub bằng credential
        - `Tag and Push`: Tag và push docker image lên dockerhub
        - `Deploy pod`: Deploy deployment với replicas=3 (3 pod) lên K8S cluster
## 9.3 Kiểm tra sau khi pipeline chạy xong
- Jenkins UI dashboard
    + ![alt text](41.png)
    + Pipeline sẽ được trigger sau khi có push event trên Gitlab repo

- Kiểm tra các Pod trên K8S cluster
    + `kubectl get deployment -n jenkins`
    + `kubectl get pods -n jenkins`
    + ![alt text](42.png)
- Kiểm tra container trong Pod
    + `kubectl describe pod my-app-6f6c67ddd5-m5ppm -n jenkins`
    + ![alt text](43.png)
- Kiểm tra log của container 
    + `kubectl logs my-app-6f6c67ddd5-m5ppm -n jenkins`
    + ![alt text](44.png)

## 10. Kết luận
- Như vậy ta đã triển khai thành công pipeline
    + Setup Jenkins
    + Setup Gitlab repo
    + Setup Docker hub
    + Setup K8S cluster (minikube)
    + Connect Jenkins vs Gitlab
    + Connect Jenkins vs Dockerhub
    + Connect Jenkins vs K8S cluster
    + Auto trigger Jenkins pipeline khi có push event từ Gitlab
    + Push image lên dockerhub
    + Deploy image lên 3 pod của K8S cluster
