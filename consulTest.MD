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
kubectl create ns plateer
helm install my-guestbook guestbook/  --namespace plateer  --values guestbook/values-consul.yaml  --wait
kubectl port-forward service/consul-ingress-gateway -n plateer 9080:8080 --address 0.0.0.0

http:<<CLIENT_IP>>:8080 접속하여 DB 쓰여지는지 확인

helm uninstall my-guestbook -n plateer 
kubectl delete pvc -l  app.kubernetes.io/instance=my-guestbook  -n plateer
kubectl delete ns plateer
```
