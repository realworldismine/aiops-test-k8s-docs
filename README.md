# aiops-test-k8s-docs

## Intial Setting

### Worker Node 연결
- kubeadm token 발급 후 join
- k8s, containerd 설정 추가 및 재시작

### 배포 관련 참고사항
- workload를 control-plain에는 절대 배포하지 말자
- kube-proxy는 한번씩 재확인하자
- coredns를 rollout해서 다시 시작하자

### Docker container 생성 및 배포
- 생각보다 많이 애먹었음
- 필요한 패키지는 apt-get update 후 설치
- python 가상환경 꼭 만들고 pip 업그레이드 하기
- wheel부터 설치
- Timezone은 UTC+0 으로 설정되어 있으므로 로컬 시간 반드시 적용
- 빌드 후 Docker hub에 배포하기

### Docker 자체 실행
- 실행 시, config 정보를 가져올 파일 지정
- 외부에서 볼 수 있도록 주요 디렉터리 매핑
- Docker 설정 내부에서 아무리 외부 DB 주소 입력해봤자, local 네트워크에서는 인식 안됨, 반드시 주소 매핑 필요

### Kubernetes 환경 구성
- 네임스페이스부터 만들자
- 워크로드 외부에서 모니터링하려면 sc, pv, pvc부터 만들 것
- 외부 DB 주소 및 포트 매핑하려면 svc, ep 만든 후, svc 이름을 cm의 호스트명에 입력할 것
- pv 만들 때 노드 어피니티 지정하고, 외부 접근을 위한 저장소를 지정하자
- pvc는 만들어서 pv와 연결해도 되고, 워크로드에서 템플릿 만들고 sc를 입력하면 파드 생성 시 sc에 있는 pv하고 연결할 pvc를 자동으로 생성함
- 외부 매핑 경로가 전혀 다를 경우 pv를 따로 만들어야 함

### 워크로드 구성
- 컨테이너 이미지는 항상 가져오는 식으로 구성(프로그램 따라 다름)
- 컨테이너 별 request, limit은 항상 설정하자
- volume mount할 때 pv별로 지정하고, 하위 경로는 subPath로 설정하자
- volume mount 시 cm도 매핑 가능, cm 변수명을 subPath로 하고 실제 경로를 매핑시킬것
- configmap에서 적용할 데이터 지정할 것
- volume 지정 시 pv 또는 cm 입력
- 노드 지정은 필요하면 반드시 할 것
