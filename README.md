# Eojin Lee (이어진)

반도체 소자 및 공정 이해를 바탕으로 RTL을 설계하고, AI Agent로 설계 검증을 자동화하는 프로젝트를 주로 진행하는 고려대학교 전기전자공학부 학부생입니다.

[![Email](https://img.shields.io/badge/Email-iuhj0519%40gmail.com-blue)](mailto:iuhj0519@gmail.com)
[![Blog](https://img.shields.io/badge/Blog-Naver-03C75A)](https://blog.naver.com/iuhj0519)

| 분야 | 하는 일 | 대표 프로젝트 |
|---|---|---|
| AI / AX | 근거가 고정된 RAG와 Gate 승인 기반 Agent Loop로 RTL 생성·검증 자동화 | [rag_db_project](https://github.com/iuhj0519-jpg/rag_db_project) |
| 회로 설계 | SystemVerilog·Chisel·Verilog 기반 NPU·연산기 RTL 설계와 FPGA PPA 분석 | [gongdae3.0_npu_study](https://github.com/iuhj0519-jpg/gongdae3.0_npu_study), [chisel_tpu_study](https://github.com/iuhj0519-jpg/chisel_tpu_study), [digital_verilog_project](https://github.com/iuhj0519-jpg/digital_verilog_project) |
| 소자·공정 | DEVSIM 기반 MOSFET 소자 설계·최적화, Fab 데이터 기반 수율 분석 | [tcad_project](https://github.com/iuhj0519-jpg/tcad_project) |

---

## AI / AX

### RAG 기반 설계 지식 DB와 Gate 승인 기반 자동화 Loop
`개인 프로젝트` · 2026.08 – 2026.09 · [rag_db_project](https://github.com/iuhj0519-jpg/rag_db_project)

승인된 SPEC과 Reference RTL만 근거로 Agent가 NPU RTL을 생성하고, 합성 병목을 탐지해 개발자 승인 후 수정·재검증까지 이어지는 Loop를 구축했습니다.

**RAG 설계 지식 DB** (2026.08.26 – 09.16)
- Markdown 명세와 RTL을 의미 단위로 Chunking하고, SQLite FTS5 BM25와 384차원 Feature Hash를 결합한 Hybrid Retrieval 구현
- 자료별 접근 정책과 원문 해시를 관리하고, 비교용 RTL은 검색 근거에서 제외해 생성 근거의 추적성 확보
- Unified Buffer·주소 재사용·AXI Prefetch를 적용해 기존 비교용 최적화 설계 대비 Buffer 논리 용량 8,140 B → 4,170 B (−48.77%)
- RAG 생성 초기 설계 대비 100개 입력 시뮬레이션 종료 시각 1,837,345 ns → 1,617,525 ns (−11.96%), MNIST 99/100 유지
- 입력 수명 기반 주소 재사용으로 용량을 줄인 대신 다음 Batch 입력 수락 시점이 제한되어, 이중 버퍼를 쓴 직접 최적화본(1,087,155 ns)보다 종료 시각은 48.79% 늘어난 저장 공간–처리 시간 절충 구조

**Gate 승인 기반 반자동화 검증·디버깅 Loop** (2026.09.02 – 09.20)
- Python 보고서 Parser와 탐지 규칙으로 메모리 매핑·LUT 집중·DSP 활용·Timing·Fan-out·전력 회귀를 분석하고 SPEC 보완안 생성
- 개발자가 SPEC과 수정 범위를 승인하면 Agent가 패치를 작성하고 Compile → Simulation·SAIF → Synthesis → Implementation → 전력 분석을 자동 실행
- Weight 저장부를 5-bank 동기 ROM 구조로 재설계해 RAMB36 10개로 매핑하고, 25개 PE에 DSP MAC 매핑

| 항목 | 수정 전 | 수정 후 |
|---|---|---|
| LUT | 18,094 | 1,199 (−93.37%) |
| WNS | −4.576 ns | +0.120 ns |
| 전력 (SAIF 기반 추정) | 0.614 W | 0.169 W (−72.48%) |
| 추론 Cycle / MNIST 예측 | 161,735 | 161,735 / 100개 결과 동일 |

자동화 회귀 테스트 56건 통과, 실패·재승인 이력과 미확정 검증 항목(I/O 제약, DRC 경고)은 후속 SPEC 안건으로 기록했습니다.

---

## 회로 설계 (RTL)

### 5×5 Systolic Array 기반 NPU 설계 및 Unified Buffer 최적화
`개인 프로젝트 · 공대 3.0 스터디` · 2026.07 – 2026.08 · [gongdae3.0_npu_study](https://github.com/iuhj0519-jpg/gongdae3.0_npu_study)

- 오픈소스 MNIST FC-MLP(784–30–20–10)에 25개 8-bit signed MAC PE로 구성한 5×5 Systolic Array NPU를 SystemVerilog로 이식
- 전체 추론 결과가 0으로 출력되는 문제를 Weight·Bias 초기화 파일 경로 오류에서 찾아 수정, ModelSim에서 MNIST 99/100 확인
- Input Buffer와 Global Buffer를 A/B 두 영역의 Unified Buffer로 통합하고, Input Loader를 분리해 현재 Batch를 계산하는 동안 다음 Batch를 수신
- 100개 입력의 시뮬레이션 종료 시각 1,844,235 ns → 1,087,155 ns (−41.05%), 정확도 99% 유지
- Vivado 합성·배치배선 결과로 메모리 접근·주소 계산·선택 회로의 개선 요소를 도출해 후속 RAG 프로젝트로 연결

### RISC-V RoCC 기반 16×16 NPU RTL 및 제어 로직 설계
`팀 프로젝트 · LZero (고려대학교 컴퓨터 아키텍처 학회)` · 2026.07 – 2026.09 · [chisel_tpu_study](https://github.com/iuhj0519-jpg/chisel_tpu_study)

- Chisel로 256개 MAC PE와 Accumulator로 구성한 16×16 Weight-Stationary Systolic Array 설계
- Data Orchestrator Unit에서 행별 0~15 cycle 지연으로 입력과 부분합의 도착 시점을 하드웨어에서 정렬
- RoCC 명령 형식 기반 64-bit rs1 Decoder와 rs2 DMA descriptor 처리부 구현, 예약 인코딩·경계 조건 테스트벤치 작성
- 6종 요청을 취합하는 Stall Generator와 2-stage Register Tree로 8개 대상에 분배하는 Stall Distributor 설계

### 8-bit 이진수 내적 계산기 RTL 설계
`팀 프로젝트 · 디지털시스템실험 Term Project · 7-segment 변환 담당` · 2023.10 – 2023.12 · [digital_verilog_project](https://github.com/iuhj0519-jpg/digital_verilog_project)

---

## 소자·공정

### DEVSIM 기반 반도체 소자 모델링 및 최적화
`개인 프로젝트 · 진행 중` · 2026.09 – · [tcad_project](https://github.com/iuhj0519-jpg/tcad_project)

- 28 nm MOSFET 최적화 논문의 접근법을 바탕으로 gate stack·doping profile·spacer 구조와 전기적 특성의 관계를 분석
- 오픈소스 DEVSIM 예제를 분석하고, 단계별 구현·수치 검증·결과 관리·버전 관리를 갖춘 재현 가능한 시뮬레이션 workflow 설계
- 소자 물리 검증 → MOSFET 전기적 파라미터 추출 → 구조 최적화의 연구 틀을 정의했으며, 현재 MOSCAP 모델링·검증 단계

### Fab 양산 데이터 분석 및 Process Integration
`개인 실습 프로젝트 · Comento` · 2025.03 – 2025.04

- Excel로 MTS 설계, FEM 기반 Targeting, 공정 산포·마진 분석과 CDF 기반 개선 효과 검증 수행
- EDS Fail과 Fab 데이터의 상관관계를 분석해 불량 원인 후보를 좁히고, 평균 이동과 산포 증가를 구분해 수율 영향 비교

---

## IT 개발

### android_subguard
바이브코딩과 프레임워크 기반의 IT 개발 프로젝트 · [android_subguard](https://github.com/iuhj0519-jpg/android_subguard)

---

## Tech Stack

| 구분 | 기술 |
|---|---|
| HDL / 언어 | SystemVerilog, Verilog HDL, Chisel, Python, C/C++ |
| EDA / 시뮬레이션 | ModelSim, Vivado, DEVSIM |
| AI | RAG (SQLite FTS5·BM25, Hybrid Retrieval), AI Agent 기반 자동화 |
| 데이터 | NumPy, Pandas, Matplotlib, Seaborn |
| 도구 | Git, Docker, Excel |
