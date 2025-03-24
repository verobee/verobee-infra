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

## ⚙️ 3. Router 설정 (Ansible Playbook)

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

---

## 4. CLOUDSTACK 설치

### CLOUDSTACK 내부 DB 설치
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/cloudstack/mysql/update_apt.yml -vvv
ansible-playbook -i ansible/inventory/hosts.ini ansible/home_infra/cloudstack/mysql/install_mysql.yml -vvv
```

## 📌 참고 사항

- **모든 시스템은 Ubuntu 24.04.2 LTS에서 구성되었습니다.**
- **Ryzen PC**는 nginx, Cloud Manager, KVM, DHCP, MariaDB, 파일 서비 등 몬스터 서비를 통합 운영하며, 내부망의 DHCP 및 NAT 중심 역할을 발표합니다.
- **N100** 노드는 CloudStack 노드 전용으로 사용하며, 경략 가상머신 운영에 적합합니다.
- 비관리형 스위치를 통해 모든 장비를 유선 연결하고, 네트워크 제어는 Ryzen PC가 단어합니다.
- 모든 방패, DHCP, NAT 설정은 Ryzen PC에서 수행됩니다.