# Azure만으로 해보는 Azure Arc 서버 실습

온프레미스 장비 없이 **Azure VM 한 대를 외부 서버처럼 준비하고 Azure Arc에 연결**합니다.
등록만 하는 데서 끝내지 않고, Arc 확장으로 VM 안에 파일을 만들어 실제 관리 동작까지 확인합니다.

> **평가·학습 전용입니다.** 일반 Azure VM은 Arc 없이도 관리할 수 있습니다.
> 이 실습은 Microsoft의 [Azure VM 평가 절차][evaluate]를 사용하며 운영용으로 지원되지 않습니다.
> 기존 업무용 VM에는 적용하지 마세요.

## 무엇을 배우나요?

```text
내 Windows PC ── SSH ── Azure Ubuntu VM 1대
                            │
                     Arc 에이전트 ── HTTPS ── Azure Arc
                                                │
                                          Portal에서 관리
```

한 서버가 Portal에 두 리소스로 보입니다. VM이 Azure 밖으로 이동하는 것은 아닙니다.

| 리소스 | 역할 |
| --- | --- |
| Azure VM: `Microsoft.Compute/virtualMachines` | 전원, CPU·메모리, 네트워크, 디스크 관리 |
| Arc 서버: `Microsoft.HybridCompute/machines` | 게스트 OS 정보, Arc 에이전트, 확장 관리 |

순서는 **VM 생성 → SSH 확인 → 평가 환경 준비 → Arc 등록 → 태그·확장 실습 → 정리**입니다.
중첩 가상화, Kubernetes, 별도 물리 서버는 필요하지 않습니다.

## 1. 시작 전 확인

### 계정과 권한

- Azure 구독과 Azure Portal 접속 권한이 필요합니다.
- 전용 리소스 그룹 생성 및 VM·네트워크·디스크 생성 권한이 필요합니다.
  `Virtual Machine Contributor`만으로 리소스 그룹·네트워크 생성까지 모두 가능한 것은 아닙니다.
- Arc 등록에는 해당 리소스 그룹의 `Azure Connected Machine Onboarding`,
  Portal에서 그룹을 선택하려면 `Reader` 권한이 필요합니다.
- 이후 태그·확장 관리와 삭제에는 `Azure Connected Machine Resource Administrator`
  또는 이를 포함하는 권한이 필요합니다. 온보딩 전용 역할만으로는 실습 전체를 수행할 수 없습니다.
- 리소스 그룹 삭제 권한도 별도로 확인합니다. 온보딩에 구독 `Owner` 계정을 사용하는 대신
  관리자가 실습용 그룹과 필요한 역할을 준비하는 방식을 권장합니다.

Portal의 **구독 → 리소스 공급자(Resource providers)** 에서 다음을 확인합니다.
등록 권한이 없다면 관리자에게 요청하고 `Registered`가 될 때까지 기다립니다.

| 공급자 | 용도 |
| --- | --- |
| `Microsoft.Compute`, `Microsoft.Network` | VM과 네트워크 |
| `Microsoft.HybridCompute` | Arc 서버와 확장 |
| `Microsoft.GuestConfiguration` | Arc 필수 공급자 |
| `Microsoft.HybridConnectivity` | Arc 필수 공급자 |

공급자 등록 자체가 유료 기능 활성화는 아닙니다.
SQL Server 실습이 아니므로 `Microsoft.AzureArcData`는 이번에 등록할 필요가 없습니다.

**구독 정책과 자동 설치도 확인하세요.** Defender for Servers, Azure Monitor,
Update Manager, Essential Machine Management 등의 플랜·정책이 이미 적용되어 있다면
추가 과금이나 확장 재설치가 발생할 수 있습니다. 강제 정책을 임의 해제하지 말고,
관리자에게 별도 실습 구독 또는 승인된 예외를 요청합니다. 조정할 수 없다면 여기서 중단합니다.

### 비용과 배포 승인

Arc의 기본 인벤토리·태그 관리와 Custom Script 실행은
[현재 기본 제어 영역 요금][pricing]에서 무료로 안내됩니다.
**실습 전체가 무료인 것은 아닙니다.** VM, OS 디스크, 공인 IPv4 주소, 일부 네트워크 사용은 과금됩니다.
모니터링·보안·업데이트 등 부가 서비스는 별도 요금이 생길 수 있어 이번 실습에서 활성화하지 않습니다.

Portal에서 다음을 먼저 확인하고 비용 부담을 승인한 뒤에만 **만들기(Create)** 를 누릅니다.

| 확인할 항목 | 확인 방법 |
| --- | --- |
| 구독과 리전 | 구독 정책·허용 리전, Arc 메타데이터 지원 리전 확인 |
| VM 크기 | 가벼운 단일 서버용 버스터블 B 계열과 일반 목적 D 계열 후보 비교 |
| 가용성과 할당량 | 선택 리전의 해당 SKU 가용성, 전체·제품군 vCPU 할당량 확인 |
| 예상 비용 | VM 생성 화면과 [가격 계산기][calculator]에서 실제 선택값 확인 |
| 실습 시간 | 사용할 시간과 종료·삭제 시점을 정하고 비용 추정에 반영 |

고정 SKU나 요금은 이 문서에서 보장하지 않습니다. Portal이 허용하는
**2 vCPU·4 GiB 수준의 작은 x64 후보부터** 비교하고, 메모리 부족 시 비용을 다시 확인합니다.
VM 가격만 보지 말고 디스크와 공인 IP도 합산하세요.
Cost Management 예산 알림은 지출을 강제로 막는 기능이 아닙니다.

## 2. 전용 Ubuntu VM 만들기

Portal의 **가상 머신 → 만들기 → Azure 가상 머신**에서 다음을 지정합니다.
Portal UI 명칭은 표시 언어와 업데이트에 따라 달라질 수 있습니다.

| 항목 | 실습 설정 |
| --- | --- |
| 리소스 그룹 | 새 전용 그룹 `rg-arc-lab-01` |
| VM 이름 | `arc-lab-ubuntu-01` |
| 이미지 | Canonical Ubuntu Server **24.04 LTS**, **x64**, 일반 이미지 |
| VM 크기 | 앞에서 비용·할당량을 확인한 작은 크기 |
| 가용성 | 단일 VM. 별도 고가용성 리소스 없음 |
| 인증 | SSH 공개 키, 사용자 `arcadmin` |
| 디스크 | 기본 OS 디스크만 사용. 추가 데이터 디스크 없음 |
| VNet·서브넷 | 실습 전용 새 VNet·서브넷 |
| 공인 IP | 실습용 Standard 공인 IP 한 개, VM NIC에 연결 |
| NIC NSG | Advanced에서 전용 NSG를 구성하거나 선택 |
| SSH 인바운드 | TCP 22, **실습자 공인 IPv4 `/32`만** 허용 |
| 부가 관리 | 모니터링·자동 패치·Defender·백업·추가 확장을 새로 활성화하지 않음 |

**SSH를 `Any` 또는 `0.0.0.0/0`에 열지 마세요.**
기본 화면의 SSH 체크박스가 전체 인터넷 허용 규칙을 만들었다면 규칙을 제거하고,
고급 NSG 설정에서 소스 IP를 제한한 뒤 배포합니다.
회사 프록시·VPN 환경에서는 실제 SSH 트래픽의 공인 출구 IP를 관리자와 확인하세요.
IP가 바뀌면 NSG와 이후 Ubuntu 방화벽의 허용 IP도 갱신해야 합니다.

아웃바운드는 NIC에 직접 연결한 공인 IP를 사용합니다. 서브넷의 기본 아웃바운드가
자동으로 제공된다고 가정하지 않습니다. 강제 터널링·UDR·방화벽이 적용되어 있다면
DNS, Ubuntu 패키지 저장소와 [Arc 필수 HTTPS 엔드포인트][network] 접근을 먼저 확인합니다.
Arc 연결만을 위해 별도의 인바운드 포트를 열 필요는 없습니다.

Trusted Launch 등 보안 기본값을 무조건 해제하지 않습니다. 다만 자동으로
Guest Attestation 등의 확장이 설치되었다면, 다음 실습의 **확장 제거 단계**에서 확인해야 합니다.
정책이 확장을 계속 재설치하면 진행하지 말고 관리자와 확인하세요.

**검토 + 만들기**에서 비용과 선택값을 다시 확인하고, 승인했다면 배포합니다.
개인 키를 내려받았다면 안전한 사용자 폴더에 보관하세요.
키·토큰과 다운로드한 온보딩 스크립트를 이 Git 저장소에 복사하거나 커밋하지 않습니다.

## 3. SSH 접속 확인

VM 배포 후 Portal에서 공인 IP를 확인합니다.
다음 명령은 **내 Windows PC의 PowerShell**에서 실행합니다.
꺾쇠 부분을 실제 값으로 바꾸세요.

```powershell
ssh -i "$HOME\.ssh\<PRIVATE_KEY_FILE>" arcadmin@<VM_PUBLIC_IP>
```

예제 경로와 다른 폴더에 키를 저장했다면 실제 키 경로를 사용합니다.
첫 접속의 호스트 키 지문은 신뢰할 수 있는 관리 경로로 확인하고 수락합니다.
시간 초과라면 VM 전원, 공인 IP, NSG의 소스 `/32`, 회사 네트워크의 TCP 22 차단부터 확인합니다.
이를 해결하려고 SSH를 전체 인터넷에 개방하지 마세요.

접속한 **Ubuntu SSH 터미널**에서 실행합니다.

```bash
cat /etc/os-release
uname -m
sudo -v
```

**통과 기준:** Ubuntu `24.04`, 아키텍처 `x86_64`, sudo 성공.
여기까지는 Azure Linux Guest Agent나 메타데이터 접근을 변경하지 않습니다.
SSH 접속이 불안정하다면 다음 단계로 넘어가지 마세요.

이제 [Arc 등록·관리·정리 실습](docs/hands-on.md)으로 이동합니다.

## 검증 상태와 참고

이 저장소는 실습 가이드입니다. **Azure 리소스를 배포해 끝까지 검증한 상태는 아닙니다.**
구독별 가격·정책·할당량과 Portal의 실제 표시값은 배포 전 확인해야 합니다.
실습을 중단해도 만든 리소스는 자동 삭제되지 않습니다.
사용하지 않을 때는 상세 가이드의 정리 절차를 수행하세요.
VM을 할당 해제해도 디스크·공인 IP 등의 비용은 남을 수 있습니다.

- [공식 평가 절차][evaluate]
- [지원 OS·권한·필수 공급자][prerequisites]
- [승인된 설계](docs/superpowers/specs/2026-11-11-azure-arc-server-lab-design.md)
- [구현 계획](docs/superpowers/plans/2026-11-11-azure-arc-server-lab.md)

[evaluate]: https://learn.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine
[prerequisites]: https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites
[network]: https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements
[pricing]: https://azure.microsoft.com/en-us/pricing/details/azure-arc/
[calculator]: https://azure.microsoft.com/en-us/pricing/calculator/
