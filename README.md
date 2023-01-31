
# Sample Helm Chart

- [Guestbook Application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)

## Step 1. 배포에 필요한 도구 설치

- Helm 을 테스트 해보기 위해서는 미리 배포된 Kubernetes Cluster 가 필요하다.
- Cluster 와 연결할 클라이언트는 아래와 같이 Docker, kubectl, helm , helm push plugin, yamllint 를 설치해 놓아야 한다.
   - Helm으로 Harbor에 Chart를 Push하기 위해서는 helm push plugin을 설치해야 push 명령어를 사용할 수 있다.

```console
#!/usr/bin/env bash

sudo yum update -y
sudo yum -y install wget tar

######################################################################
# Install docker
######################################################################
sudo amazon-linux-extras install -y docker
sudo tee  /etc/docker/daemon.json > /dev/null <<EOF
{
"insecure-registries" : ["$private_ip","0.0.0.0"]
}
EOF
sudo service docker start
sudo usermod -a -G docker ec2-user
sudo chmod 666 /var/run/docker.sock

######################################################################
# Install docker compose
######################################################################
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

######################################################################
# Install Kubectl
######################################################################
sudo curl -L "https://dl.k8s.io/release/v1.23.6/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
sudo ln -s /usr/local/bin/kubectl /usr/bin/kubectl

######################################################################
# Install Helm
###################################################################### 
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

######################################################################
# Install helm push plugin
######################################################################
sudo yum -y install git
helm plugin install https://github.com/chartmuseum/helm-push

######################################################################
# Install yamlint 
######################################################################
sudo yum install -y python3-pip
sudo pip3 install yamllint
yamllint --version
```

## Step 2. Cluster 와 연결

```console
aws configure
aws eks --region ap-northeast-2 update-kubeconfig --name <CLUSTER_NAME>
chmod 600  ~/.kube/config
```
