
## Sample Helm Chart

- [Guestbook Application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)

### Step 1. 배포에 필요한 도구 설치

- Helm 을 테스트 해보기 위해서는 미리 배포된 Kubernetes Cluster 가 필요하다.
- Cluster 와 연결할 클라이언트는 아래와 같이 Docker, kubectl, helm , helm push plugin, yamllint 를 설치해 놓아야 한다.
   - 아래는 ec2 아마존 리눅스 2 인스턴스에 이와 같은 도구를 설치하는다.
   - Helm으로 Harbor에 Chart를 Push하기 위해서는 helm push plugin을 설치해야 push 명령어를 사용할 수 있다.

```console
#!/usr/bin/env bash

private_ip=$( curl -Ss -H "X-aws-ec2-metadata-token: $imds_token" 169.254.169.254/latest/meta-data/local-ipv4 )

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

### Step 2. Cluster 와 연결

```console
aws configure
aws eks --region ap-northeast-2 update-kubeconfig --name <CLUSTER_NAME>
chmod 600  ~/.kube/config
```


### Reference - Helm 주요 커맨드 및 환경 변수 

- 레포지토리 관리
	- 차트 레포지토리 추가
		- helm repo add  
		- helm repo add $REPO_NAME $REPO_URL
	- 차트 레포지토리 조회
		- helm repo list 
	- 차트 레포지토리 삭제
		- helm repo remove 
		- helm repo remove $REPO_NAME 
	- 차트 레포지토리에서 사용 가능한 차트에 대한 정보를 리모트에서 로컬로 업데이트
		- helm repo update 
	- 패키징된 차트를 포함하는 디렉토리가 지정된 인덱스 파일 생성
		- helm repo index 
- 플러그인 관리
	- 플러그인 인스톨
		- helm plugin install 
		- helm plugin install $URL
	- 설치된 플러그인 조회
		- helm plugin list
	- 한 가지 이상의 플러그인 삭제
		- helm plugin uninstall
		- helm plugin uninstall $PLUGIN
	- 한 가지이상의 플러그인 업데이트
		- helm plugin update
		- helm plugin update $PLUGIN
- 환경변수
	- XDG_CACHE_HOME 
		- 설치된 차트 정보 캐싱
	- XDG_CONFIG_HOME 
		- 레포지토리 정보 저장
		- helm repo add 명령을 실행하여 추가된 레포지토리 정보를 저장
	- XDG_DATA_HOME 
		- 플러그인 저장
	- HELM_DRIVER
		- release state 를 어떻게 저장할지 여부
		- 설정 가능한 값
			- secret (default) > state 를 base64 로 인코딩후 k8s 시크릿으로 저장
			- configmap > state 를 일반 텍스트로 저장하여 k8s 컨피그맵으로 저장
	- HELM_NO_PLUGINS
		- 플러그인 비활성화
		- 설정 가능한 값
			- 1 > 비활성화
			- 0 (default) > 활성화 
	- KUBECONFIG
		- 헬름이 쿠버네티스를 사용하기 위해 인증이 필요하며 이를 설정하기 위한 쿠버네티스 설정 파일 경로
		- 기본값 >  ~/.kube/config
	- HELM_NAMESPACE
		- 헬름이 상호작용해야 하는 네임스페이스를 나타내는 환경변수
	- helm env
		- 설정된 헬름 환경변수 확인

