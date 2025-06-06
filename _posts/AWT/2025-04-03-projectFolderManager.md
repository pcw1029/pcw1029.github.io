---
layout: single
title:  "회사 폴더 구성 설명"
date: 2025-04-03
tags: etc
categories: 
   - AWT
---

# 폴더 구성

## Proposal (제안서 및 초기 기획)

프로젝트를 시작하기 위한 기획 단계 문서로서 투자, 개발 승인, 연구 개발(R&D) 제안 등에 필요

### 1. Project_Proposal (프로젝트 제안서)

   1. 프로젝트 배경 및 필요성
   2. 목표 및 기대 효과
   3. 기능 개요
   4. 예상 개발 기간 및 비용
   5. 경쟁 제품 분석 및 차별점

   

### 2. Technical_Review (기술 검토 문서)

   1. 유사 프로젝트 분석
   2. 현재 기술 수준 검토
   3. 사용 가능한 하드웨어 및 소프트웨어 선택 기준

   

### 3. Development_Schedule (개발 일정)

   1. 프로젝트 진행 일정 (Gantt Chart 포함)
   2. 주요 마일스톤

   

## Requirements (요구 사항 분석)

고객 또는 프로젝트 팀이 정의한 요구 사항을 정리
하드웨어, 펌웨어, 소프트웨어의 요구 사항을 모두 포함

### 1. System_Requirements (요구 사항 정의서)

   1. 기능 요구사항(기능 설명, 성능 요구사항)
   2. 비기능 요구 사항(전력 소모, 크기, 내구성 등..)
   3. 하드웨어, 펌웨어, 소프트웨어 별 요구 사항 정의

   

### 2. Functional_Requirements (기능 명세서)

   1. 각 기능의 동작 방식 및 처리 방식 정의

   

### 3. Interface_Requirements (인터페이스 요구 사항)

   1. 시스템 간 통신 요구 사항 (프로토콜, 속도 등...)
   2. UI 및 사용자 인터페이스 요구 사항



## Design (설계)

프로젝트 설계 문서를 저장하는 폴더
시스템의 아키텍처 및 구조를 정의

### 1. System_Architecture (시스템 아키텍처 문서)

   1. 전체 시스템 개요
   2. 하드웨어, 펌웨어, 소프트웨어 간 관계 및 인터페이스

   

### 2. Hardware_Design (하드웨어 설계 문서)

   1. 회로 다이어그램, 부품 리스트(BOM)
   2. PCB 설계 기준 및 제작 지침

   

### 3. Firmware_Design (펌웨어 설계 문서)

   1. 펌웨어 소프트웨어 구조 (Task, Thread, ISR 등)
   2. 주요 모듈 및 데이터 흐름도

   

### 4. Software_Design (소프트웨어 설계 문서)

   1. GUI 구조, 통신 방식 정의
   2. 데이터 처리 방식 (예: 이미지 처리, 센서 데이터 분석)

   

### 5. Interface_Control_Doucument (ICD 문서)

   1. 하드웨어-펌웨어, 펌웨어-소프트웨어 간의 데이터 교환 규격
   2. 통신 프로토콜, 데이터 포맷 정의



## Development (개발)

프로젝트의 실제 개발 작업이 이루어지는 공간으로, 하드웨어(HW), 소프트웨어(SW), 펌웨어(FW)의 소스 코드 및 관련 파일들이 포함됩니다. 이 영역은 개발자들이 가장 자주 사용하는 폴더로, 개발 도중 수시로 변경되거나 확장될 수 있습니다.

### 1. 하드웨어 (HW)

FPGA 및 회로 관련 개발을 위한 폴더 구조로, 논리 설계, IP, 제약 조건 파일 등 모든 하드웨어 구현 자료를 포함합니다.

- `Rtl` : 주요 RTL 설계 파일들 (Verilog/VHDL 등)
- `Packages` : Vivado 등에서 생성된 package 파일
- `Testbenchs` : 시뮬레이션 및 검증용 테스트벤치 코드
- `IP-BDs` : Block Design 기반의 IP 설계
- `IP` : 직접 설계하거나 타사에서 제공받은 IP 모듈
- `Constraints` : 타이밍, 핀 배치 등의 제약 조건 파일
- `ExportHwPlatform` : SDK 또는 Vitis로 내보내는 하드웨어 플랫폼 파일
- `Scripts` : 빌드 자동화 및 작업 편의를 위한 스크립트
- `Tcl` : Vivado용 Tcl 스크립트 모음
- `Docs` : 하드웨어 관련 개발 문서



### 2. 소프트웨어 (SW)

임베디드 리눅스, 애플리케이션, 드라이버 등의 소프트웨어 개발을 위한 영역입니다. 구조는 프로젝트에 따라 유동적이므로, 기본적으로 개발 문서만 포함됩니다.

- `Docs` : 소프트웨어 개발 관련 설명서 및 참고 문서

※ 애플리케이션 소스, 스크립트, 빌드 시스템 등은 프로젝트 초기 혹은 중간에 구성되며, 프로젝트의 요구에 따라 폴더가 자유롭게 확장됩니다.



### 3. 펌웨어 (FW)

MCU 기반 제어 로직 또는 펌웨어 설계를 위한 공간입니다. 마찬가지로 구조는 프로젝트 성격에 따라 자유롭게 구성되며, 초기에는 문서 중심으로 관리됩니다.

- `Docs` : 펌웨어 구조 및 기능 설계 문서

※ 펌웨어 소스 코드 및 빌드 환경은 개발 진행 상황에 따라 구성됩니다.



## Docs (최종 기술 문서)

프로젝트 완료 후 필요한 문서

### 1. Final_Report (최종 보고서)
### 2. Maintenance (유지보수 매뉴얼 )
### 3. Presentations (프레젠테이션 자료)
### 4. User_Manual (가이드 메뉴얼)



## Archive (완료된 프로젝트 보관)

과거 문서 및 프로젝트 백업

### 1. 프로젝트 종료 보고서
### 2. 이전 버전의 주요 문서
