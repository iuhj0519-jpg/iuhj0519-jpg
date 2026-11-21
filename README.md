# Eojin Lee (이어진)

반도체 소자 및 공정 이해를 바탕으로 RTL을 설계하고, AI Agent로 설계 검증을 자동화하는 프로젝트를 주로 진행하는 고려대학교 전기전자공학부 학부생입니다.

[![Email](https://img.shields.io/badge/Email-iuhj0519%40gmail.com-blue)](mailto:iuhj0519@gmail.com)
[![Naver](https://img.shields.io/badge/Email-iuhj0519%40naver.com-03C75A)](mailto:iuhj0519@naver.com)
[![Korea University](https://img.shields.io/badge/Email-iuhj0519%40korea.ac.kr-8B0029)](mailto:iuhj0519@korea.ac.kr)
[![Blog](https://img.shields.io/badge/Blog-blog.naver.com%2Fiuhj0519-03C75A)](https://blog.naver.com/iuhj0519)

| 분야 | 대표 프로젝트 |
|---|---|
| AI / AX | [rag_db_project](https://github.com/iuhj0519-jpg/rag_db_project) |
| 회로 설계 | [gongdae3.0_npu_study](https://github.com/iuhj0519-jpg/gongdae3.0_npu_study), [chisel_tpu_study](https://github.com/iuhj0519-jpg/chisel_tpu_study), [digital_verilog_project](https://github.com/iuhj0519-jpg/digital_verilog_project) |
| 소자 / 공정 | [tcad_project](https://github.com/iuhj0519-jpg/tcad_project) |

---

## AI / AX

### [RAG 기반 설계 지식 DB와 Gate 승인 자동화 Loop](https://github.com/iuhj0519-jpg/rag_db_project)
개인 프로젝트 | 2026.08 – 2026.09

LLM에 그때그때 물어서 만든 RTL은 어떤 근거로 작성됐는지 추적하기 어려웠습니다. 그래서 승인된 SPEC과 Reference RTL만 검색되도록 접근 정책과 원문 해시를 관리하는 로컬 RAG를 만들고, 기존 비교용 RTL은 근거에서 빼서 Agent가 이 자료만으로 NPU를 다시 설계하게 했습니다. 검색은 SQLite FTS5 BM25와 384차원 Feature Hash를 섞은 Hybrid 방식입니다.

RAG로 만든 최적화 설계는 입력 주소를 재사용해 직접 최적화한 설계보다 Buffer 논리 용량을 48.77% 줄였고(8,140 B → 4,170 B), 자체 초기 설계보다 시뮬레이션 종료 시각을 11.96% 앞당겼습니다. 다만 다음 Batch 입력을 받는 시점이 늦어져, 이중 버퍼를 쓴 직접 최적화본(1,087,155 ns)보다는 48.79% 느립니다. 저장 공간과 처리 시간을 맞바꾼 구조입니다.

이어서 합성 결과를 Agent가 스스로 점검하는 Loop를 붙였습니다. Python Parser가 Vivado 보고서에서 메모리 매핑, LUT 집중, DSP 활용, Timing 문제를 찾아 SPEC 보완안을 만들고, 제가 승인한 범위만 Agent가 수정한 뒤 Compile부터 Implementation, SAIF 전력 분석까지 다시 돌립니다. 승인된 수정으로 Weight 저장부를 RAMB36 10개로, 25개 PE의 MAC을 DSP로 옮긴 결과는 아래와 같습니다.

| 항목 | 수정 전 | 수정 후 |
|---|---|---|
| LUT | 18,094 | 1,199 (−93.37%) |
| WNS | −4.576 ns | +0.120 ns |
| 전력 (SAIF 기반 추정) | 0.614 W | 0.169 W (−72.48%) |
| 추론 Cycle | 161,735 | 161,735 |

MNIST 100개 입력의 예측 결과는 그대로 유지했고, 회귀 테스트 56건을 통과했습니다. I/O 제약과 DRC 경고는 아직 풀지 못해 다음 SPEC 검토 항목으로 남겨 두었습니다.

`Python` `SQLite FTS5` `RAG` `SystemVerilog` `Vivado` `ModelSim`

---

## 회로 설계

### [5×5 Systolic Array NPU와 Unified Buffer 최적화](https://github.com/iuhj0519-jpg/gongdae3.0_npu_study)
개인 프로젝트 (공대 3.0 스터디) | 2026.07 – 2026.08

오픈소스 MNIST FC-MLP(784–30–20–10)에 8-bit signed MAC 25개로 이루어진 5×5 Systolic Array를 이식했습니다. 처음에는 모든 추론 결과가 0으로 나왔는데, 데이터 흐름을 따라가다 Weight·Bias 초기화 파일 경로가 잘못된 것을 찾아 고쳤고 이후 100개 중 99개를 맞혔습니다.

분석해 보니 매 Batch마다 AXI 입력 로딩이 연산 앞에 그대로 더해지고 있었습니다. Input Buffer와 Global Buffer를 A/B 두 영역의 Unified Buffer로 합치고 Input Loader를 연산 제어에서 떼어내, 현재 Batch를 계산하는 동안 다음 Batch를 받도록 바꿨습니다. 100개 입력의 시뮬레이션 종료 시각은 1,844,235 ns에서 1,087,155 ns로 41.05% 줄었습니다.

| 항목 | 원본 FC-MLP | 5×5 Systolic Array | 최적화 후 |
|---|---|---|---|
| Batch 크기 | 1 | 5 | 5 |
| 100개 입력 시뮬레이션 종료 시각 | – | 1,844,235 ns | 1,087,155 ns (−41.05%) |
| 이미지당 평균 cycle | 약 873.2 | 약 1,844.2 | 약 1,087.2 |
| Batch당 cycle 구성 | – | AXI 입력 3,920 + 연산 5,301 | 첫 Batch 이후 AXI 입력을 연산과 중첩 |
| Buffer 구조 | – | Input Buffer + Global Buffer | A/B Unified Buffer (814 depth × 5 × 2) |
| MNIST 정확도 | 99% | 99% | 99% |

`SystemVerilog` `ModelSim` `Vivado`

### [RISC-V RoCC 기반 16×16 NPU RTL](https://github.com/iuhj0519-jpg/chisel_tpu_study)
팀 프로젝트 (LZero, 고려대학교 컴퓨터 아키텍처 학회) | 2026.07 – 2026.09

Chisel로 256개 MAC PE를 가진 16×16 Weight-Stationary Systolic Array를 만들고, Data Orchestrator가 행마다 0~15 cycle씩 지연을 줘서 입력과 부분합이 제때 만나도록 했습니다. 제어 쪽에서는 RoCC 명령의 64-bit rs1을 해석하는 Decoder와 rs2 DMA descriptor 처리부, 6종의 정지 요청을 모아 2단 Register Tree로 8곳에 나눠 주는 Stall 제어 로직을 맡았습니다.

`Chisel` `Scala` `RISC-V RoCC`

### [8-bit 이진수 내적 계산기](https://github.com/iuhj0519-jpg/digital_verilog_project)
팀 프로젝트 (디지털시스템실험 Term Project), 7-segment 변환 담당 | 2023.10 – 2023.12

---

## 소자 / 공정

### [DEVSIM 기반 반도체 소자 모델링과 최적화](https://github.com/iuhj0519-jpg/tcad_project)
개인 프로젝트, 진행 중 | 2026.09 –

28 nm MOSFET 최적화 논문을 읽고, 같은 접근으로 gate stack, doping profile, spacer 구조가 전기적 특성을 어떻게 바꾸는지 직접 시뮬레이션해 보고 있습니다. 오픈소스 DEVSIM 예제를 분석한 뒤 단계별 구현, 수치 검증, 결과 관리를 갖춘 재현 가능한 workflow를 짰고, 지금은 MOSCAP 모델링과 검증 단계에 있습니다.

`Python` `DEVSIM` `Gmsh`

### Fab 양산 데이터 분석과 Process Integration
개인 실습 프로젝트 (Comento) | 2025.03 – 2025.04

Excel로 MTS 설계와 FEM 기반 Targeting을 하고, EDS Fail과 Fab 데이터의 상관관계를 따라가며 불량 원인 후보를 좁혔습니다. 평균이 움직인 경우와 산포가 커진 경우를 구분해서 보니, 같은 평균이라도 규격 여유에 따라 수율 판단이 달라진다는 것을 확인할 수 있었습니다.

`Excel` `CDF` `FEM`

---

## IT 개발

### [android_subguard](https://github.com/iuhj0519-jpg/android_subguard)
바이브코딩과 프레임워크 기반의 IT 개발 프로젝트

---

## Tech Stack

| 구분 | 기술 |
|---|---|
| HDL / 언어 | SystemVerilog, Verilog HDL, Chisel, Python, C/C++ |
| EDA / 시뮬레이션 | ModelSim, Vivado, DEVSIM |
| AI | RAG, AI Agent 기반 설계 자동화 |
| 데이터 | NumPy, Pandas, Matplotlib, Seaborn |
| 도구 | Git, Docker, Excel |
