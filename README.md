# 🐝 Verobee Home Infra Setup Guide

Verobee 프로젝트를 위한 **홈 인프라 기반 CloudStack IaaS 환경** 구체 가이드입니다.  
본 가이드는 **Ubuntu 24.04.1 LTS** 기반에서 모든 시스템을 구성하였으며, macOS에서는 Ansible을 통해 네트워크 및 서비스 자동화 설정을 수행합니다.  
CloudStack Manager, KVM 노드, MariaDB, nginx 등을 통합하여 실행 가능한 프라이브트 클라우드 환경을 구성합니다.

---

## 💡 현실적인 저전량 홈 IaaS 환경 구성

> 에너지 효율성과 공간 효율을 고려한 실제 홈래브 장비 구성을 기반으로 합니다.

### 장비 구성

| 구리       | 사양                                      | 역할 설명 |
|------------|-------------------------------------------|------------|
| Ryzen PC   | Ryzen 5825U, RAM 64GB, SSD 1TB           | 메인 서버: nginx, CloudStack Manager, KVM, DHCP, MariaDB, 파일 서버 운영 |
| N100       | Intel N100, RAM 16GB, SSD 512GB          | CloudStack Node로 사용 (하이퍼바이저) |
| 스위치 허브 | 비관리형 8포트 기가비트 스위치           | 네트워크 허브 – 모든 장비 유선 연결, 네트워크 할당은 Ryzen PC에서 수행 |

- Ryzen PC는 모든 주요 서비스를 통합 운영하며, 내부망의 중심 역할을 수행합니다.
- N100 장비는 경략 가상화를 위한 하이퍼바이저 노드로 활용됩니다.
- 스위치는 단순한 물리적 연결만 제공하며, 네트워크 할당 및 제어는 Ryzen PC에서 단당합니다.

---

## 📦 1. macOS 필수 설치 프로그램

```bash
brew install ansible
brew install sshpass
```

---

## 🌐 2. 네트워크 구성도

| 외부 망 (Internet) | ⇄ | Ryzen PC *(5825U)* | ⇄ | Switch Hub | ⇄ | 내부 Infra (KVM Nodes 등) |
|-------------------|----|---------------------|----|-------------|----|---------------------------|
|                   |     | - nginx              |     |             |     | - KVM Node (N100)         |
|                   |     | - Cloud Manager      |     |             |     |                           |
|                   |     | - KVM Hypervisor     |     |             |     |                           |
|                   |     | - DHCP Server        |     |             |     |                           |
|                   |     | - MariaDB            |     |             |     |                           |
|                   |     | - 파일 서버          |     |             |     |                           |

---

## ⚙️ 3. 네트워크 설정 (Ansible Playbook)

내부망 구성을 위한 네트워크 설정은 아래 순서대로 실행하세요:

| 순서 | Playbook 경로 | 설명 |
|------|----------------|------|
| 1 | `setup_internal_network.yml` | 내부 인터피스 및 브리지 구성 |
| 2 | `setup_iptables.yml`        | NAT 및 포워딩 설정 (외부 ↔ 내부 연결) |
| 3 | `setup_dhcp.yml`            | 내부망을 위한 DHCP 서버 구성 |
| 4 | `setup_ufw.yml`             | UFW 방패용 설정 (필수 포트 허용 등) |

### 실행 명령어

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/router/update_apt.yml -vvv
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/router/setup_internal_network.yml -vvv
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/router/setup_dhcp.yml -vvv
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/router/setup_iptables.yml -vvv
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/router/setup_ufw.yml -vvv
```

> ⚠️ `--check`는 실제 적용 없이 실행 예시입니다. 실제 적용 시에는 제거하세요.

---

## 🌐 4. 서비스 설치 (Nginx / MariaDB)

### nginx 설치

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/nginx/install_nginx.yml -vvv
```

### MariaDB 설치

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/mariadb/install_mariadb.yml -vvv
```

> 이 서비들은 CloudStack Manager가 사용하는 DB 및 웹 UI 서비입니다.

---

## ☁️ 5. CloudStack 구성 단계

### 🔧 CloudStack Manager 설치

- Ryzen PC (**Ubuntu 24.04.1 LTS**) 기반에 설치
- 외부 MariaDB와 연동
- CloudStack Web UI 및 API 구독 확인

### 💻 CloudStack Node 설치 및 연결

- N100 장비에 KVM 기반 하이퍼바이저 설치
- CloudStack Manager에 Host 등록
- 가상 네트워크 구성

### 🚀 CloudStack CKS (Container Service) 설정

- CloudStack의 Kubernetes Service 활성화
- 기본 클러스터 구성 및 테스트

---

## 📁 디렉토리 구조 예시

```bash
ansible/
├── inventory/
│   └── hosts.ini
├── home_infra/
│   ├── proxy/
│   │   ├── setup_internal_network.yml
│   │   ├── setup_iptables.yml
│   │   ├── setup_dhcp.yml
│   │   └── setup_ufw.yml
│   ├── nginx/
│   │   └── install_nginx.yml
│   └── mariadb/
│       └── install_mariadb.yml
```

---

## 📌 참고 사항

- **모든 시스템은 Ubuntu 24.04.1 LTS에서 구성되었습니다.**
- **Ryzen PC**는 nginx, Cloud Manager, KVM, DHCP, MariaDB, 파일 서비 등 몬스터 서비를 통합 운영하며, 내부망의 DHCP 및 NAT 중심 역할을 발표합니다.
- **N100** 노드는 CloudStack 노드 전용으로 사용하며, 경략 가상머신 운영에 적합합니다.
- 비관리형 스위치를 통해 모든 장비를 유선 연결하고, 네트워크 제어는 Ryzen PC가 단어합니다.
- 모든 방패, DHCP, NAT 설정은 Ryzen PC에서 수행됩니다.