## Test

values.yaml 사용

```console 
git clone https://github.com/kin3303/guestbook.git
kubectl create ns plateer
helm install my-guestbook guestbook/  --namespace plateer  --wait
kubectl port-forward service/my-guestbook -n plateer  8080:80 --address 0.0.0.0

http:<<CLIENT_IP>>:8080 접속하여 DB 쓰여지는지 확인

helm uninstall my-guestbook -n plateer 
kubectl delete pvc -l  app.kubernetes.io/instance=my-guestbook  -n plateer
kubectl delete ns plateer
```

values-consul.yaml 사용

```console 
git clone https://github.com/kin3303/guestbook.git

# 배포
kubectl create ns plateer
helm install my-guestbook guestbook/  --namespace plateer  --values guestbook/values-consul.yaml  --wait

# 배포된 릴리즈 확인
$ helm ls 
$ helm get values <RELEASE_NAME>

# 서비스 변경시 재시작 
 kubectl rollout restart deploy/consul-ingress-gateway -n consul
 kubectl rollout restart deploy/my-guestbook -n plateer
 kubectl rollout restart statefulset/my-guestbook-redis-master -n plateer
 kubectl rollout status deploy/consul-ingress-gateway --watch -n consul
 kubectl rollout status deploy/my-guestbook --watch -n plateer
 kubectl rollout status statefulset/my-guestbook-redis-master --watch -n plateer 
 
# 서비스 확인 
 kubectl port-forward service/consul-ingress-gateway -n consul 9080:8080 --address 0.0.0.0
 kubectl port-forward service/consul-server --namespace consul 8501:8501 --address 0.0.0.0

 http:<<CLIENT_IP>>:8080
 https://<CLIENT_IP>:8501/ui 

# 릴리즈 
helm uninstall my-guestbook -n plateer 
kubectl delete pvc -l  app.kubernetes.io/instance=my-guestbook  -n plateer
kubectl delete ns plateer
```
