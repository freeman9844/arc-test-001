# Azure 안에서 시작하는 Azure Arc 서버 실습 설계

> **과거 승인 기록:** 아래는 최초 Ubuntu 실습 설계입니다. 이후 사용자 요청으로
> Windows Server 기반으로 변경했습니다. 현재 실습은 [README](../../../README.md)와
> [상세 가이드](../../hands-on.md)를 따르며 아래 Ubuntu 명령·설정을 적용하지 않습니다.

## 목적과 승인 상태

Azure 외부 장비가 없는 사용자가 Azure 구독만으로 Azure Arc-enabled servers의
등록과 기본 관리를 직접 경험한다.
사용자는 서버 관리부터 작게 시작하는 방향과 Ubuntu 한 대 설계안을 승인했다.
이 승인은 설계에 대한 승인이다. Azure 리소스 생성과 비용 발생은 별도 승인 대상이다.

## 접근 방식

1. **전용 Azure VM을 외부 서버처럼 구성:** 선택한 방식이다. VM 한 대로 시작하며
   Microsoft가 문서화한 평가 전용 절차를 사용한다.
2. **VM 위 Kubernetes 클러스터를 Arc에 연결:** GitOps 실습에 적합하지만 이번 서버
   관리 입문 목적에는 불필요한 구성과 학습 부담이 생겨 제외했다.
3. **Jumpstart ArcBox:** 여러 Arc 기능을 함께 체험하기에 좋지만 이번 범위보다 크고
   리소스와 비용 부담이 커서 제외했다.

## 구성

```text
사용자 PC ── SSH(실습자 공인 IP만 허용) ── 전용 Ubuntu VM
                                             │
                                    Arc 에이전트의 아웃바운드 HTTPS
                                             │
                                     Azure Arc 서버 리소스
                                             │
                                       Azure Portal 관리
```

- 전용 리소스 그룹에 Ubuntu 24.04 LTS(x64) VM 한 대, VNet, 서브넷, NSG,
  NIC, OS 디스크와 SSH용 공인 IP를 둔다.
- 중첩 가상화는 사용하지 않는다. 기존 업무용 VM은 변경하지 않는다.
- Azure VM은 가상 하드웨어를, 별도 `Microsoft.HybridCompute/machines` 리소스는
  게스트 OS를 관리한다. VM이 Azure 밖으로 이동하는 것은 아니다.
- Portal로 리소스와 Arc 등록 스크립트를 준비하고, 사용자 PC의 SSH로 Ubuntu를 설정한다.
  이는 구현자가 제안한 입문용 도구 선택이다. 별도 IaC 도구와 배포 자동화는 도입하지 않는다.
- 구독, 리전, VM 크기는 실제 배포 전에 가용성·할당량·예상 비용을 확인하여 확정한다.
  현재 특정 구독의 배포 가능성이나 요금을 검증한 상태는 아니다.

## 실습 범위와 완료 기준

1. VM 생성과 SSH 접속을 확인한다.
2. 기존 Azure VM 확장 제거, Azure Linux Guest Agent 비활성화, Azure IMDS 접근 차단,
   `MSFT_ARC_TEST=true` 설정으로 공식 평가 전용 준비 절차를 수행한다.
3. Connected Machine Agent를 설치하고 사용자 대화형 인증으로 Arc에 등록한다.
4. Portal과 `azcmagent show`에서 연결 상태, OS, 에이전트 정보를 확인한다.
5. Arc 서버 리소스에 태그를 추가하고 조회한다.
6. Linux용 Custom Script 확장으로 `/var/tmp/arc-lab-check.txt`에 실습 문자열을
   기록하고 SSH에서 확인한다. 확장을 제거한 뒤 제거 상태도 확인한다.
7. Arc 연결 해제 후 실습 리소스 그룹을 삭제하고 Arc 리소스·VM·디스크·공인 IP의
   잔존 여부를 확인한다.

Arc `Connected` 상태와 확장을 통한 실제 파일 생성이 핵심 성공 기준이다.
태그 변경만으로 게스트 OS 관리까지 검증했다고 간주하지 않는다.
설치할 확장의 Ubuntu 24.04 지원과 별도 과금 여부는 문서 작성 시 확인한다.

## 안전·비용 제약

- 이 구성은 평가·테스트 전용이며 Azure VM의 운영 구성으로 지원되지 않는다.
- Guest Agent 비활성화 후 Azure VM Run Command와 기존 VM 확장에 의존하지 않는다.
  SSH 접속을 먼저 검증하며, 설정 실패 시 새 실습 VM으로 다시 시작할 수 있게 안내한다.
- SSH 인바운드는 실습자 공인 IP의 `/32`로 제한한다. Arc용 인바운드 포트는 열지 않는다.
  공인 IP 변경 시 SSH 허용 규칙도 갱신한다.
- Arc 통신과 설치 패키지에 필요한 아웃바운드·DNS는 유지한다.
  IMDS 차단을 인터넷 차단으로 확대하지 않는다.
- 방화벽 적용 전에 SSH 허용 규칙을 준비한다. `169.254.169.254`와
  `169.254.169.253` 차단 및 재부팅 후 상태 재검증을 안내한다.
- VM·디스크·공인 IP 등은 과금될 수 있다. VM 할당 해제만으로 모든 과금이
  중단되지는 않는다. 예상 비용과 실습 시간을 확인한 후 배포 승인을 받는다.
- Azure Monitor, Defender for Servers, Update Manager 등 추가 과금 가능 기능을
  실습에서 새로 활성화하지 않는다. 구독 정책에 의한 자동 설치·과금 가능성도 사전 점검한다.
- Kubernetes, SQL Server, Private Link, Arc Gateway, 운영 고가용성은 범위에서 제외한다.
- 키·토큰·개인 식별 정보와 다운로드한 온보딩 스크립트는 저장소에 넣지 않는다.

## 구현자에게 맡기는 사항

Portal 화면의 최신 명칭, 최소 권한 안내, VM 후보의 실제 가격·할당량 확인,
Custom Script 확장의 정확한 설정과 버전, 단계별 실패 진단 명령을 문서로 구체화한다.
실제 배포를 수행하지 않았다면 문서를 실제 환경에서 검증했다고 표현하지 않는다.

## 근거 문서

- [Azure VM을 이용한 Arc 평가](https://learn.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine)
- [에이전트 필수 조건과 지원 OS](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites)
- [Portal 온보딩](https://learn.microsoft.com/en-us/azure/azure-arc/servers/onboard-portal)
- [네트워크 요구 사항](https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements)
- [Arc 확장 관리](https://learn.microsoft.com/en-us/azure/azure-arc/servers/manage-vm-extensions)
- [ArcBox](https://jumpstart.azure.com/azure_jumpstart_arcbox)
