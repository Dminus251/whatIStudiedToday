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

# Argo CD란?
- Argo CD는 **선언적인** 쿠버네티스의 깃옵스 CD 도구다.
- Argo CD의 핵심 구성 요소는 애플리케이션 컨트롤러.
- 애플리케이션 컨트롤러가 애플리케이션을 지속적으로 관찰하여 깃 리포지토리와 상태를 일치시킨다.<br><br>

### 조정(reconciling)
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
