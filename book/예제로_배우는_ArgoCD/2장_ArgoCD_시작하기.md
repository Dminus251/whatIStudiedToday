# Argo CD가 왜 필요할까?

### 전통적으로 환경을 분리하는 방법
- dev, test, staging, production **환경 별로 cluster나 name space를 분리한다.**
- 이 방법은 ***구성 드리프트***가 발생한다.
- **구성 드리프트**: 시간이 지남에 따라 인프라 자원들이 서로 다른 상태가 되는 현상
  - 예를 들어 특정 클러스터나 네임 스페이스에서만 최신 버전의 애플리케이션이 분리됨<br><br>
- 따라서 변경 사항이 있을 경우, 분리된 모든 환경의 리소스를 수정해야 하므로 리소스를 관리하기 어렵다.<br><br>

### 깃옵스 접근 방식
- 깃 리포지터리에 모든 원천 소스를 보관한다.
- 1장에서 설명한 컨트롤러가 깃 리포지터리의 모든 구성을 자동으로 적용하는 방법

<br>

# Argo CD란?
- Argo CD는 **선언적인** 쿠버네티스의 깃옵스 CD 도구다.
- Argo CD의 핵심 구성 요소는 애플리케이션 컨트롤러.
- 애플리케이션 컨트롤러가 애플리케이션을 지속적으로 관찰하여 깃 리포지토리와 상태를 일치시킨다.<br><br>

## 조정(reconciling)
![제목 없음](https://github.com/user-attachments/assets/4912db3b-d3a6-4e2f-bf4f-6a1e7f011d05)<br>
사진 출처: https://subscription.packtpub.com/book/cloud-and-networking/9781803233321/3/ch03lvl1sec15/explaining-architecture<br><br>
깃 리포지터리에 담긴 의도한 상태를 현재 상태의 클러스터와 일치시키는 과정을 ***조정***이라고 한다.<br>
1. Argo CD는 깃 리포지터리의 헬름 차트를 쿠버네티스 yaml로 렌더링한다. ($ helm template)
2. 이 렌더링된 yaml을 클러스터 상태와 비교한다.
3. Argo CD가 다르다고 판단하면 kubectl apply를 사용해 쿠버네티스를 의도한 상태로 변경한다.

# Argo CD 아키텍처
![download](https://github.com/user-attachments/assets/d2c87aff-4362-4626-9555-ba77453a96a2)<br>
사진 출처: https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/<br><br>
- **API Server**
  - 웹 UI, CLI, Argo Event, CI/CD 시스템과 같이 다른 시스템들과 API를 통해 상호작용한다.
- **Repository Server**
  - 애플리케이션 매니페스트를 보관하는 깃 리포지터리의 로컬 캐시를 유지한다.
- **Application Controller**
  - 지속적으로 애플리케이션의 현재 상태를 확인하고, 깃 리포지터리의 상태와 비교한다.
  - 생명 주기 동안 동기화 훅을 실행한다.

# ArgoCD 핵심 오브젝트와 리소스
### Application
Argo CD는 'Application'이라는 CRD를 이용해 애플리케이션 인스터스를 구현한다. 예를 들어 다음과 같다.
```
apiVersion: argoproj.io/v1alpha1
kind: Application #CRD
metadata:
  name: guiestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://~~~~.git
    targetRevisioN: HEAD
    path: guestbook
destination:
  server: https://kubernetes.default.svc
  namepsace: guestbook
```

### App project
앱 프로젝트 CRD는 태그 지정과 같이, 유관한 애플리케이션들을 논리적으로 그룹화할 수 있다. 

### Repository Credentials
실제 운영에선 private repository를 사용하기 때문에, Argo CD가 이 레포지터리에 접근하기 위해 자격 증명이 필요하다.
Kubernetes Secret과 ConfigMap 리소스를 사용해 이 문제를 해결한다.
```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namepsace: argocd
  labels:
    argocd.argoproj.io/secrett-type: repository
  stringData:
    url: git@github.com:argoproj/my-private-repository
    sshPrivateKey: |
    -----BEGIN OOPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

# 첫 Application을 배포해 보자.
kind로 예제 cluster yaml을 apply해도 되지만 내 경우 minikube를 사용함
```
helm repo add argo https://argoproj..github.io/argo-helm
kubectl create namespace argocd
helm install ch02 --namepsace argocd argo/argo-cd
kubectl port-forward service/ch02-argocd-server -n argocd 8080:443 #localhost의 8080 포트를 서비스의 443 포트로 포워딩
```
<br>

이제 argo CD UI에 접근할 수 있고, username은 admin, password는 아래 명령의 결과로 로그인할 수 있다.
```
 k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

<br>

아래 application CRD의 yaml 파일을 배포하자
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx #application 이름
  namespace: argocd #argo CD applicaation이 배포될 ns
  finalizers:
    #아래 finalizer는 ArgoCD가 application을 삭제할 때 리소스를 안전하게    #제거하도록 보장함
    - resources-finalizer.argocd.argoproj.io
spec:
  syncPolicy:
    automated: #자동 동기화 정책 설정
      prune: true #cluster에 존재하지만 git에 정의되지 않은 리소스 자동 제거
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  destination:
    namespace: nginx #application이 배포될 ns
    server: https://kubernetes.default.svc #cluster API 서버의 도메인
    #참고: https://stackoverflow.com/questions/54048632/how-does-the-kubernetes-default-svc-routed-to-the-api-server
  project: default #deafult project에서 application 관리
  source:
    repoURL: https://charts.bitnami.com/bitnami #application source의 주소. 
    chart: nginx #chart name
    targetRevision: 13.2.10 #chart versiond
```
<br>
이제 UI로 application을 확인할 수 있다.

![image](https://github.com/user-attachments/assets/b6c2537f-4357-401c-8b4c-eaeb3c4b4d38)

<br>
# Argo CD Auto Pilot
![image](https://github.com/user-attachments/assets/5a8185a5-6e7a-4289-82fc-fed2268456c4)
사진 출처: https://argocd-autopilot.readthedocs.io/en/stable/
<br>

# 동기화의 원리
### 동기화란?
쿠버네티스 클러스터에 변경 내용을 적용해 애플리케이션을 타깃 상태로 만드는 단계.
동기화는 다음과 같은 단계를 거쳐 실행된다.
1. 사전 동기화 (pre-sync)
2. 동기화 (sync)
3. 사후 동기화(post-sync)

이들을 ***리소스 훅***이라고 하며, 각 단계에서 다른 작업을 실행할 수 있는 권한을 제공한다.

### 리소스 훅
- PreSync: 동기화 단계 전에 완료돼야 하는 작업 ex) db migration
- Sync: 롤링 업데이트 전략보다 더 복잡한 배포를 오케스트레이션할 때 사용
- PostSync: 배포 후에 통합 상태를 확인하는 등에 사용
- 그 외 Skip, SyncFail도 있다.
예를 들어 아래와 같이 Job resource와 annotation을 통해 사용할 수 있다.
```
apiVersion: batch/v1
kind: Job
metadata:
  generateName: presync-job
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: presync-job
          image: ubuntu
          command:
            - /bin/bash
            - -c
            - |
              echo "Pre Sync Job"
      restartPolicy: Never
  backoffLimit: 2
```
<br>

### 동기화 웨이브
Sync Wave는 여러 리소스를 동기화할 때 실행 순서를 제어하는 기능이다. 
기본값은 0이며, 음수와 양수값 모두 사용 가능하다.
wave 값이 낮은 리소스부터 동기화가 시작된다.
아래 예시처럼 annotation을 간단하게 추가하면 사용할 수 있다.
```
metadata:
  annocations:
    argocd.argoproj.io/sync-wave: "5"
```
