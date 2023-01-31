
## Sample Helm Chart

- [Guestbook Application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)

### Step 1. 배포에 필요한 도구 설치

- Helm 을 테스트 해보기 위해서는 미리 배포된 Kubernetes Cluster 가 필요하다.
- Cluster 와 연결할 클라이언트는 아래와 같이 Docker, kubectl, helm , helm push plugin, yamllint 를 설치해 놓아야 한다.
   - 아래는 ec2 아마존 리눅스 2 인스턴스에 이와 같은 도구를 설치하는다.
   - Helm으로 Harbor에 Chart를 Push하기 위해서는 helm push plugin을 설치해야 push 명령어를 사용할 수 있다.

```console 
$  private_ip=$( curl -Ss -H "X-aws-ec2-metadata-token: $imds_token" 169.254.169.254/latest/meta-data/local-ipv4 )
$  sudo yum update -y
$  sudo yum -y install wget tar

######################################################################
# Install docker
######################################################################
$  sudo amazon-linux-extras install -y docker
$  sudo tee  /etc/docker/daemon.json > /dev/null <<EOF
{
"insecure-registries" : ["$private_ip","0.0.0.0"]
}
EOF
$  sudo service docker start
$  sudo usermod -a -G docker ec2-user
$  sudo chmod 666 /var/run/docker.sock

######################################################################
# Install docker compose
######################################################################
$  sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$  sudo chmod +x /usr/local/bin/docker-compose
$  sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

######################################################################
# Install Kubectl
######################################################################
$  sudo curl -L "https://dl.k8s.io/release/v1.23.6/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
$  sudo chmod +x /usr/local/bin/kubectl
$  sudo ln -s /usr/local/bin/kubectl /usr/bin/kubectl

######################################################################
# Install Helm
###################################################################### 
$  curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

######################################################################
# Install helm push plugin
######################################################################
$  sudo yum -y install git
$  helm plugin install https://github.com/chartmuseum/helm-push

######################################################################
# Install yamlint 
######################################################################
$  sudo yum install -y python3-pip
$  sudo pip3 install yamllint
$  yamllint --version
```

### Step 2. Cluster 와 연결

```console
$  aws configure
$  aws eks --region ap-northeast-2 update-kubeconfig --name <CLUSTER_NAME>
$  chmod 600  ~/.kube/config
```

### Step 3. Chart 정적 테스트

- helm lint 명령을 통해 Chart 에 이상이 없는지 검사 

```console
$  helm lint guestbook 
==> Linting guestbook
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- yaml lint 명령을 통해 Chart Yaml 에 이상이 없는지 검사

```console
$ git clone https://github.com/kin3303/guestbook.git

######################################################################
# yamllint 설정
######################################################################
$ sudo tee  .yamllint > /dev/null <<EOF
extends: default
rules:
  line-length:
    max: 150
  trailing-spaces:
    level: warning
  key-duplicates:
    level: error
EOF

######################################################################
# yamllint 실행
######################################################################
$  helm template my-guestbook guestbook --namespace gs  | yamllint -
```

### Step 4. Chart 패키지 및 Harbor 에 업로드

- 먼저 Harbor 레포지토리에 guestbook 이라는 프로젝트를 만든다.
- Chart 패키징 및 패키징된 Chart 를 Harbor 에 업로드 한다. (아래 코드 참조)

```console
######################################################################
# Add Repository 
######################################################################
$ helm repo add guestbook-repo https://<YOUR_HARBOR_DOMAIN>/chartrepo/guestbook --username=<HARBOR_USER_NAME> --password=<HARBOR_PASSWORD>

######################################################################
# Package Helm Chart 
######################################################################
$  ls
guestbook
$ helm package guestbook/
Successfully packaged chart and saved it to: /home/ec2-user/helm_test/guestbook-0.1.0.tgz
$  ls
guestbook guestbook-0.1.0.tgz

######################################################################
# Upload Helm Chart 
######################################################################
$  helm cm-push guestbook-0.1.0.tgz  guestbook-repo --username=<HARBOR_USER_NAME> --password=<HARBOR_PASSWORD>
 
```

### Step 5. Chart 설치 및 테스트

- 먼저 Harbor 레포지토리에 guestbook 이라는 프로젝트를 만든다.
- Chart 패키징 및 패키징된 Chart 를 Harbor 에 업로드 한다. (아래 코드 참조)

```console
######################################################################
# 레포지터리 업데이트 및 확인
######################################################################
$ cd ..
$ mkdir helm-install-test
$ cd helm-install-test
$  helm repo update
$ helm search repo guestbook-repo -l 
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                
guestbook-repo/guestbook        0.1.0           5.0.0          A Helm chart for Kubernetes

######################################################################
# 차트 설치
######################################################################
$ kubectl create ns gs
$  helm install my-guestbook guestbook-repo/guestbook --version 0.1.0 --namespace gs  --wait
$ kubectl port-forward service/my-guestbook -n gs  8080:80 --address 0.0.0.0
http://<호스트_IP>:8080 으로 접속하여 데이터 입력

######################################################################
# 동적 테스트
######################################################################
$ helm test my-guestbook -n gs --logs
NAME: my-guestbook
LAST DEPLOYED: Tue Jan 31 14:16:01 2023
NAMESPACE: gs
STATUS: deployed
REVISION: 1
TEST SUITE:     my-guestbook-test-backend-connection
Last Started:   Tue Jan 31 14:23:17 2023
Last Completed: Tue Jan 31 14:23:21 2023
Phase:          Succeeded
TEST SUITE:     my-guestbook-test-frontend-connection
Last Started:   Tue Jan 31 14:23:21 2023
Last Completed: Tue Jan 31 14:23:25 2023
Phase:          Succeeded
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace gs -o jsonpath="{.spec.ports[0].nodePort}" services my-guestbook)
  export NODE_IP=$(kubectl get nodes --namespace gs -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

POD LOGS: my-guestbook-test-backend-connection
,aa,bb,cc

POD LOGS: my-guestbook-test-frontend-connection
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
<html ng-app="redis">
  <head>
    <title>Guestbook</title>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.12/angular.min.js"></script>
    <script src="controllers.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular-ui-bootstrap/2.5.6/ui-bootstrap-tpls.js"></script>
  </head>
  <body ng-controller="RedisCtrl">
    <div style="width: 50%; margin-left: 20px">
      <h2>Guestbook</h2>
    <form>
    <fieldset>
    <input ng-model="msg" placeholder="Messages" class="form-control" type="text" name="input"><br>
    <button type="button" class="btn btn-primary" ng-click="controller.onRedis()">Submit</button>
    </fieldset>
    </form>
    <div>
      <div ng-repeat="msg in messages track by $index">
        {{msg}}
      </div>
    </div>
    </div>
  </body>
</html>
100   920  100   920    0     0   224k      0 --:--:-- --:--:-- --:--:--  224k

######################################################################
# 리비전 업그레이드
######################################################################
$ kubectl exec -n gs -it my-guestbook-redis-master-0  --  redis-cli -h localhost save
$ kubectl scale statefulsets/my-guestbook-redis-master  --replicas=0 -n gs
$ kubectl scale statefulsets/my-guestbook-redis-replicas --replicas=0 -n gs
$ kubectl get pods -n gs
$ helm upgrade  my-guestbook guestbook-repo/guestbook -n gs   --wait  
$ kubectl exec -n gs -it my-guestbook-redis-master-0 -- /bin/sh
DB 백업 확인
$ kubectl port-forward service/my-guestbook   -n gs  8080:80 --address 0.0.0.0
데이터 추가 입력

######################################################################
# 롤백 테스트
######################################################################
$ kubectl scale statefulsets/my-guestbook-redis-master  --replicas=0 -n gs
$ kubectl scale statefulsets/my-guestbook-redis-replicas --replicas=0 -n gs
$ kubectl get pods -n gs
$ helm rollback my-guestbook 1 -n gs   --wait  
$ kubectl port-forward service/my-guestbook   -n gs  8080:80 --address 0.0.0.0
롤백 확인

######################################################################
# 리소스 정리
######################################################################
$ helm uninstall my-guestbook -n gs 
$ kubectl delete pvc -l  app.kubernetes.io/instance=my-guestbook  -n gs
$ kubectl delete ns gs
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

