---
layout: single
title:  "Verilog RTL/Simulation 관리용 Visual Studio Code 환경 구성"
date: 2025-04-03
tags: etc
categories: 
   - AWT
---

# Verilog RTL/Simulation 관리용 Visual Studio Code 환경 구성

이 프로젝트는 Vivado RTL 개발 환경 안에서 Verilog RTL 및 Testbench 코드를 Visual Studio Code로 관리하고, 간단한 시뮬레이션과 파형 확인까지 수행할 수 있도록 구성한 것이다.




---

## 폴더 구조

HW/ ├── Rtl/                    # Verilog RTL 코드 

​	 ├── Testbenchs/            # 테스트벤치 코드 

​	 ├── project_plasmaSensor/  # Vivado 프로젝트 

​	 ├── ... 

​	 └── vs_code/               # ✅ VS Code 전용 환경 

​		├── .vscode/           # VS Code 설정 (tasks.json, settings.json) 

​		├── sim/               # (선택) 시뮬레이션 스크립트 

​		├── build/             	# 시뮬레이션 결과 저장 

​		├── filelist.f			 # 포함할 Verilog 파일 목록 

​		├── Makefile			# 빌드 자동화 

​		├── code-workspace	 # 워크스페이스 파일 





---

## 필수 VS Code 확장 (Plugin)

| Plugin 이름               | 기능                        |
| ------------------------- | --------------------------- |
| ✅ Verilog HDL (by mshr-h) | 문법 강조, lint             |
| Verible Formatter         | Google SystemVerilog 포매터 |
| Code Runner               | 선택 코드 실행              |
| Better Comments           | 주석 강조                   |
| SystemVerilog (선택)      | SystemVerilog 사용 시       |
| Icarus Verilog            | 무료 Verilog 시뮬레이터     |
| ✅ GTKWave                 | 파형 분석 GUI 툴            |

---



## 워크스페이스 파일 생성

**파일명:** `plasmaSensor.code-workspace`  
**위치:** `HW/vs_code/`

```json
{
  "folders": [
    {
      "path": "."
    },
    {
      "path": "../Rtl"
    },
    {
      "path": "../Testbenchs"
    }
  ],
  "settings": {
    "verilog.linting.linter": "iverilog",
    "verilog.linting.iverilog.arguments": ["-Wall", "-g2012"],
    "files.associations": {
      "*.v": "verilog",
      "*.sv": "systemverilog"
    }
  }
}

필요 시 ../IP-BDs, ../Tcl, ../Scripts 등도 추가 가능
```



## VS Code 설정 (`.vscode/settings.json`)

```json
{
  "verilog.linting.linter": "iverilog",
  "verilog.linting.iverilog.arguments": ["-Wall", "-g2012"],
  "files.associations": {
    "*.v": "verilog",
    "*.sv": "systemverilog"
  },
  "editor.tabSize": 2,
  "editor.detectIndentation": false
}
```

---



## filelist.f – 소스 파일 목록

`filelist.f`에는 시뮬레이션에 포함할 RTL 및 Testbench 파일을 명시합니다.

```
../Rtl/top.v
../Rtl/alu.v
../Rtl/mux.v
../Testbenchs/tb_top.v
```

---



## Task 자동 실행 설정 (`.vscode/tasks.json`)

```
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Simulate with filelist",
      "type": "shell",
      "command": "iverilog -o ../build/test.out -f ../filelist.f && vvp ../build/test.out",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": []
    }
  ]
}
```

> `Ctrl+Shift+B`로 시뮬레이션 실행

---



## Makefile (선택)

```
TOPMODULE=tb_top
OUT=build/test.out
DUMP=build/dump.vcd
SRC=$(shell cat filelist.f)

all: $(OUT)
	@vvp $(OUT)

$(OUT): $(SRC)
	@mkdir -p build
	iverilog -o $(OUT) -g2012 -f filelist.f

wave: all
	gtkwave $(DUMP)

clean:
	rm -rf build/*
```

**사용 예시:**

```
make        # 시뮬레이션 실행
make wave   # GTKWave로 파형 보기
make clean  # 빌드 파일 삭제
```

---



## GTKWave 설치 (Ubuntu)

```
sudo apt update
sudo apt install gtkwave
```

파형 확인:

```
gtkwave build/dump.vcd
```



---

## Icarus Verilog (iverilog)란?

**Icarus Verilog**는 Verilog RTL 및 테스트벤치 코드를 **시뮬레이션하기 위한 오픈소스 컴파일러**입니다.  
Vivado 없이도 간단히 RTL 코드의 동작을 빠르게 검증할 수 있습니다.

### 왜 필요한가요?

| 목적        | 설명                                               |
| ----------- | -------------------------------------------------- |
| 시뮬레이션  | Verilog 소스와 테스트벤치를 컴파일하여 동작을 검증 |
| 버그 확인   | 실제 회로 구현 전에 논리 오류/시나리오 검토        |
| 파형 보기   | `$dumpvars`와 `GTKWave`를 통해 결과 시각화         |
| 빠른 테스트 | Vivado 없이도 가볍게 코딩 후 검증 가능             |

---

### 동작 방식

```bash
iverilog -o build/test.out -f filelist.f   # 1. 컴파일
vvp build/test.out                         # 2. 시뮬레이션 실행
gtkwave build/dump.vcd                     # 3. 파형 보기 (옵션)
```



### Icarus Verilog 설치 방법 (Ubuntu 기준)

```bash
sudo apt update
sudo apt install iverilog
```
