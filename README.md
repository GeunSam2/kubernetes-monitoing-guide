# Kubernetes + Monitoring

Kubernetes 메트릭을 이용한 리소스 모니터링에는 Grafana 대시보드, Log 모니터링은 Kibana 대시보드를 사용한다.

# Metric Monitoring

**PreRequire**

처음 1회만 작업이 필요하다. API 사용자는 PreRequire를 스킵한다.(Token 값은 사전에 전달)

kubernetes API를 통해 pod의 이름등을 알기 위해서 전체 관리자 권한이 부여된 서비스 계정을 생성해야한다.

```yaml
kubectl apply -f admin-access.yaml
```

```yaml
#yaml for admin account create

kind: ServiceAccount
apiVersion: v1
metadata:
  name: root-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: root-sa-kube-system-cluster-admin
subjects:
- kind: ServiceAccount
  name: root-sa
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

생성한 서비스계정의 토큰값을 받아온다.

```bash
ROOTTOKEN="$(kubectl get secret -nkube-system $(kubectl get secrets -nkube-system | grep root-sa | cut -f1 -d ' ') -o jsonpath='{$.data.token}' | base64 --decode)"
```

**모니터링 대상 컨테이너 이름 탐색**

label 및 서비스이름 정보 활용하여 kubernetes api를 통해 서비스 pod의 이름을 호출해야 한다.

ex) aihub 네임스페이스에서 {'app' : 'test-nginx'} 라는 label을 가진 pod을 찾는 api 구문

```bash
#bash
curl  -X GET --insecure --header "Authorization: Bearer $ROOTTOKEN" https://192.168.0.230:6443/api/v1/namespaces/aihub/pods?labelSelector="app%3Dtest-nginx"
```

```bash
#result example

{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/aihub/pods",
    "resourceVersion": "29661716"
  },
  "items": [
    {
      "metadata": {
        "name": "openldap-6c799f5984-xww56", #we need here!
        "generateName": "openldap-6c799f5984-",
        "namespace": "aihub",
########################
------- 중간 생략 ------
########################
        "qosClass": "BestEffort"
      }
    }
  ]
}
```

**페이지 호출**

```bash
#URL
http://<grafanaURL>/d/U5I6e2iMk/kubernetes-compute-resources-pod-name1?orgId=1&refresh=10s&var-datasource=default&var-cluster=&var-namespace=<namespace 이름>&var-pod=<획득한 pod 이름>
```

```bash
#예시
http://192.168.0.230:32553/d/U5I6e2iMk/kubernetes-compute-resources-pod-name1?orgId=1&refresh=10s&var-datasource=default&var-cluster=&var-namespace=aihub-platform&var-pod=waih-ai-platform-api-download-54846bf8c5-d9fks
```

페이지 예시

![Kubernetes%20Monitoring%20e456668c5b4f4000affa4dcb4824dd5e/Untitled.png](Kubernetes%20Monitoring%20e456668c5b4f4000affa4dcb4824dd5e/Untitled.png)

# Log Monitoring

페이지 호출

pod의 이름을 이용해 패이지를 호출할 수 있다.

```bash
#URL
http://192.168.0.230:31423/app/logs/stream?flyoutOptions=(flyoutId:!n,flyoutVisibility:hidden,surroundingLogsId:!n)&logFilter=(expression:%27kubernetes.pod_name:<pod 이름>,kind:kuery)&logPosition=(end:now,position:(tiebreaker:101162,time:1591756514040),start:now-1d,streamLive:!f)
```

```bash
#예시
http://192.168.0.230:31423/app/logs/stream?flyoutOptions=(flyoutId:!n,flyoutVisibility:hidden,surroundingLogsId:!n)&logFilter=(expression:%27kubernetes.pod_name:aihub-portal-gitlab-app-5cffdc8f98-9f47k%27,kind:kuery)&logPosition=(end:now,position:(tiebreaker:101162,time:1591756514040),start:now-1d,streamLive:!f)
```

![Kubernetes%20Monitoring%20e456668c5b4f4000affa4dcb4824dd5e/Untitled%201.png](Kubernetes%20Monitoring%20e456668c5b4f4000affa4dcb4824dd5e/Untitled%201.png)