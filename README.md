
## Sample Helm Chart
 
[Guestbook Application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/) 차트의 구성

- Chart.yaml
	- 레디스 디펜던시 추가
- charts/
	- 다운로드한 레디스 디펜던시 (아래 명령으로 다운로드 가능하지만 레포지토리가 없어지거나 할 수 있어서 그냥 저장함)
	        -  helm repo update
	        -  helm dependency update guestbook 
- templates/
	- deployment.yaml
		- 방명록 어플리케이션을 배포
		- https://kubernetes.io/docs/tutorials/stateless-application/guestbook/#creating-the-guestbook-frontend-deployment
	- ingress.yaml
		- 클러스터 외부에서 방명록 어플리케이션에 액세스할 수 있는 옵션 제공
	- serviceaccount.yaml
		- 방명록 어플리케이션을 위한 전용 서비스 계정을 생성
	- service.yaml
		- 방명록 어플리케이션의 여러 인스턴스 간의 부하 분산에 사용
	- _helpers.tp
		- 헬름 차트 전반에 걸처 사용될 공통 템플릿 집합을 제공
	- NOTES.txt

### Step 1. 배포에 필요한 도구 설치

- Helm 을 테스트 해보기 위해서는 미리 배포된 Kubernetes Cluster 가 필요하다.
- Cluster 와 연결할 클라이언트는 아래와 같이 Docker, kubectl, helm , helm push plugin, yamllint 를 설치해 놓아야 한다.
   - 아래는 ec2 아마존 리눅스 2 인스턴스에 이와 같은 도구를 설치하는다.
   - Helm으로 Harbor에 Chart를 Push하기 위해서는 helm push plugin을 설치해야 push 명령어를 사용할 수 있다.
- k8s 클러스터에는 Consul, Prometheus, Grafana, Jaeger 를 미리 설치해 놓아야 한다.

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

######################################################################
# Download Sources
######################################################################
$  git clone https://github.com/kin3303/guestbook.git
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
$ ls
guestbook
$  helm lint guestbook 
==> Linting guestbook
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- yaml lint 명령을 통해 Chart Yaml 에 이상이 없는지 검사

```console
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
$ helm template my-guestbook guestbook --namespace plateer  | yamllint -
  41:18     warning  trailing spaces  (trailing-spaces)
  271:1     warning  trailing spaces  (trailing-spaces)
  341:1     warning  trailing spaces  (trailing-spaces)
  450:1     warning  trailing spaces  (trailing-spaces)
  456:1     warning  trailing spaces  (trailing-spaces)
  468:1     warning  trailing spaces  (trailing-spaces)
  600:1     warning  trailing spaces  (trailing-spaces)
  606:1     warning  trailing spaces  (trailing-spaces)
  618:1     warning  trailing spaces  (trailing-spaces)
$ cat -n <(helm template my-guestbook guestbook --namespace plateer)
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
$ mkdir guestbook-from-harbor
$ cd guestbook-from-harbor
$ helm repo update
$ helm search repo guestbook-repo -l 
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                
guestbook-repo/guestbook        0.1.0           5.0.0          A Helm chart for Kubernetes

######################################################################
# 차트 설치
######################################################################
$ kubectl create ns plateer
$ helm install my-guestbook guestbook-repo/guestbook --version 0.1.0 --namespace plateer  --wait
$ kubectl get all -n plateer
$ kubectl port-forward service/my-guestbook -n plateer  8080:80 --address 0.0.0.0
http://<호스트_IP>:8080 으로 접속하여 데이터 입력

######################################################################
# 동적 테스트
######################################################################
$ helm test my-guestbook -n plateer --logs
NAME: my-guestbook
LAST DEPLOYED: Tue Jan 31 14:16:01 2023
NAMESPACE: plateer
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
  export NODE_PORT=$(kubectl get --namespace plateer -o jsonpath="{.spec.ports[0].nodePort}" services my-guestbook)
  export NODE_IP=$(kubectl get nodes --namespace plateer -o jsonpath="{.items[0].status.addresses[0].address}")
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
$ kubectl exec -n plateer -it my-guestbook-redis-master-0  --  redis-cli -h localhost save
$ kubectl scale statefulsets -l app.kubernetes.io/instance=my-guestbook --replicas=0 -n plateer
$ kubectl get pods -n plateer
$ helm upgrade  my-guestbook guestbook-repo/guestbook -n plateer --wait  
$ kubectl exec -n plateer -it my-guestbook-redis-master-0 -- /bin/sh
DB 백업 확인
$ kubectl port-forward service/my-guestbook -n plateer 8080:80 --address 0.0.0.0
데이터 추가 입력

######################################################################
# 롤백 테스트
######################################################################
$ kubectl scale statefulsets -l app.kubernetes.io/instance=my-guestbook --replicas=0 -n plateer
$ kubectl get pods -n plateer
$ helm rollback my-guestbook 1 -n plateer --wait  
$ kubectl port-forward service/my-guestbook -n plateer  8080:80 --address 0.0.0.0
롤백 확인

######################################################################
# 리소스 정리
######################################################################
$ helm uninstall my-guestbook -n plateer 
$ kubectl delete pvc -l  app.kubernetes.io/instance=my-guestbook  -n plateer
$ kubectl delete ns plateer
```

### Step 6. Guestbook with consul

``` console 
######################################################################
#  Consul 확인
######################################################################
$ kubectl port-forward service/consul-server --namespace consul 8501:8501 --address 0.0.0.0
https://<CLIENT_IP>:8501/ui 

######################################################################
# Promethues 확인
######################################################################
$ kubectl port-forward deploy/prometheus-server 9090:9090 --address 0.0.0.0
http://<CLIENT_IP>:9090  

######################################################################
# Grafana 확인
######################################################################
$ kubectl port-forward svc/grafana 3000:3000 --address 0.0.0.0
http://<CLIENT_IP>:3000 (admin/password)

######################################################################
# Jaeger 확인
######################################################################
$ kubectl port-forward svc/jaeger-query 16686:16686 --address 0.0.0.0   
http://<CLIENT_IP>:16686 

######################################################################
# Guestbook 배포
######################################################################
$ kubectl create ns plateer
$ helm install my-guestbook guestbook -n plateer  -f guestbook/values-consul.yaml --wait

######################################################################
# IngressGateway 에서 접근
######################################################################
$ kubectl get ingressgateway ingress-gateway -n consul  >> SYNCED == true 확인
$ kubectl port-forward service/consul-ingress-gateway -n consul 8080:8080 --address 0.0.0.0
```


### Reference - Chart 구조

``` 
$ cd mychart
$ tree
.
├── charts # 헬름 차트가 의존하고 있는 차트가 포함된 디렉토리
├── Chart.yaml # Chart Version 등을 담고 있는 Chart 메타데이터
├── templates # Termplate Rendering Engine 을 통해 생성할 Go 템플릿를 포함하는 디렉토리
│   ├── deployment.yaml # 샘플 리소스
│   ├── _helpers.tpl # Utility 함수를 저장
│   ├── hpa.yaml # 샘플 리소스
│   ├── ingress.yaml # 샘플 리소스
│   ├── NOTES.txt # 차트 설치 과정에서 사용 지침을 제공하기 위한 파일
│   ├── serviceaccount.yaml # 샘플 리소스
│   ├── service.yaml # 샘플 리소스
│   └── tests
│       └── test-connection.yaml
└── values.yaml # 차트의 기본값을 포함하고 있는 파일
└── values.schema.json # 차트의 기본값에 대한 스키마를 JSON 형식으로 포함하고 있는 파일
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

### Reference - Helm Hook

| 어노테이션 값 | 설명 |
|----------|----------|
| pre-install | helm install , 템플릿 렌더링후 리소스가 생성되기 전에 실행된다. |
| post-install | helm install , 쿠버네티스에 모든 리소스가 로드된 후에 실행된다. |
| pre-delete | helm uninstall , 쿠버네티스에서 리소스가 삭제되기 전에 실행된다. |
| post-delete | helm uninstall , 릴리스의 모든 리소스가 삭제된 후 실행된다. |
| pre-upgrade | helm upgrade, 템플릿 렌더링후 리소스가 업데이트되기 전에 실행된다. |
| post-upgrade | helm upgrade, 모든 리소스가 업그레이드된 후 실행된다. |
| pre-rollback | helm rollback , 템플릿 렌더링후 아무 리소스도 롤백되지 않은 시점에 실행된다. |
| post-rollback | helm rollback , 모든 리소스가 수정된 후에 실행된다. |
| test | helm test, 해당 명령이 호출될 때 실행된다. |

- 헬름은 install/upgrade/rollback/test 시 템플릿 렌더링 후 쿠버네티스 리소스를 배포하는 수명 주기를 가진다.
- 이때 헬름 어노테이션과 k8s 의 Job 을 이용하여 훅을 설정할 수 있다.
- 예를 들어 helm install 시 헬름 릴리즈의 수명주기는 다음과 같다.
	1. 사용자가 helm install foo 를 실행한다.
	2. 헬름 라이브러리 설치 API가 호출된다.
	3. 검증 후, 라이브러리는 foo 템플릿을 렌더링한다.
	4. 라이브러리는 결과로 나온 리소스를 쿠버네티스에 로드한다.
	5. 라이브러리는 릴리스 객체(및 다른 데이터)를 클라이언트에 반환한다.
	6. 클라이언트가 종료된다.
- 이때 pre-install 및 post-install 의 두 훅을 정의하면  헬름 릴리즈의 수명주기는 아래와 같이 변경된다.
	1. 사용자가 helm install foo 를 실행한다.
	2. 헬름 라이브러리 설치 API가 호출된다.
	3. crds/ 디렉터리의 CRD가 설치된다.
	4. 검증 후 라이브러리는 foo 템플릿을 렌더링한다.
	5. 라이브러리는 pre-install   훅 리소스 로딩한다.
		- 라이브러리는 가중치(기본값으로는 0을 할당), 리소스 종류, 이름을 기준으로 훅을 오름차순으로 정렬한다.
		- 라이브러리는 가장 낮은 가중치의 훅을 먼저 로드한다.(음수에서 양수로)
	6. 라이브러리는 훅이 "Ready" 될 때까지 대기한다(CRD 제외).
	7. 라이브러리는 결과로 나온 리소스를 쿠버네티스에 로드한다.
	8. 라이브러리는 post-install  훅 리소스 로딩한다.
		- --wait 플래그가 설정된 경우 라이브러리는 모든 리소스가 "Ready" 상태가 될 때까지 대기하고, 준비가 될 때까지 post-install 훅을 실행하지 않는다.
	9. 라이브러리는 훅이 "Ready" 될 때까지 기다린다.
	10. 라이브러리는 릴리스 객체(및 다른 데이터)를 클라이언트에 반환한다.
	11. 클라이언트가 종료된다.
- Hook 삭제 정책
	- helm.sh/hook-delete-policy 어노테이션을 통해서 훅을 정의한 Job 을 삭제할 시기를 결정할 수 있다.
		- before-hook-creation
			- 훅이 시작되기 전에 이전 훅을 삭제
		- hook-succeeded
			- 훅이 성공적으로 실행된 후 훅을 삭제
		- hook-failed
			- 실행 중 훅이 실패하면 훅을 삭제

### Reference - Chart Template 함수

- helm 템플릿에서 사용가능한  전체 [함수 목록](https://helm.sh/ko/docs/chart_template_guide/function_list/)

