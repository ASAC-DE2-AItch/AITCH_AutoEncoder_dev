# 능동형 제조 AI 플랫폼 기획서 v4.5

## Etch FDC Trace 기반 불량 예측 · Smart SPC 이상 감지 · R2R 능동 보정(Recipe·실력치·모델) · RAG 기반 AI Agent 의사결정 지원 통합 플랫폼

**ASAC · SK HYNIX · T ACADEMY**

**SK하이닉스 / 이주학 TL**

**2026. 07 · Ver 4.4**

> **v4.3 개정 요지 (멘토 피드백 반영)**: 이상 대응 수단을 "Recipe 보정(R2R)"에서 **"FDC 실력치(관리 기준선) 보정 + 정비 조치 + Wafer 스크랩 판정"**으로 전환. 승인 상태머신 LangGraph 구현, 인터락·의심 윈도우·Disposition, Incident 단위 처리, Qdrant 전환, 재학습 가드레일 반영.
>
> **v4.4 개정 요지 (팀 리뷰 반영)**: ① SHAP-Nelson 분리(모델 근거와 통계 근거의 독립 채널화), ② Pipeline B Autoencoder 확정, ③ Pipeline C=판정·기록 / E=원인 추론의 역할 경계 명확화, ④ VM → C65 Proxy Metrology 명명(CD 라벨 부재의 정직한 반영), ⑤ E를 fast-path(동기·규칙)/slow-path(비동기·RAG+LLM)로 분리한 7-Layer 아키텍처, ⑥ CT 재학습 2종 분리(LightGBM 자동 / Autoencoder Qual 검증+승인) 및 PM 후 Qual Wafer 검증 절차 신설, ⑦ 시연 시나리오 2종 확정.
> **v4.5 개정 요지 (멘토 정정 반영 — 2026-07-06)**: ① **Recipe R2R 부활** — v4.3의 "Recipe 보정 전면 금지"는 멘토 발언의 오해였음이 확인됨. 정정된 원칙: **무승인 자동 적용만 금지**, SHAP 기반 Recipe Tuning 제안 → 엔지니어 승인 → 적용(시뮬레이터 반영) → 효과 검증의 전체 루프를 서비스에 포함 (현업은 Agent가 레거시 DB API로 R2R을 자동 진행하나, 본 시스템은 HITL 게이트 유지 — 차별점). ② **Model R2R 신설** — PM Cycle 초반 실측-예측 격차(Residual)만큼 Bias를 모델에 자동 update (상한 clamp + 기록, 초과분은 Incident 신호). ③ **CT① 재정의** — 트리거 = PM Cycle(C33 리셋 간격) 종료, sliding window(최근 3개월 상당) 재학습, 3 Cycle 예측. ④ **Supervisor 4지선다** (Recipe Tuning 옵션 추가). ⑤ **신호-조치 라우팅 경계** — 모델 신호(SHAP)는 Recipe 제안으로, 실력치는 SPC 통계로만. ⑥ wafer 라우팅 룰(crazy wafer 1장=HOLD→승인 스크랩 / 연속=R2R / 연속 심각=RTD+BM), R² 지표 추가, RAG 개정(Vision LLM 페이지 청킹·Qwen 임베딩·사례 1만+건), C33=RF time 정정.
> **v4.6 개정 요지 (사이클 정의 확정 + 데이터 재검증 — 2026-07-08)**: **PM 사이클 세계관의 단일 소스는 `docs/사이클_정의_v1.md`** (시뮬레이터·CT·Model R2R·실력치·Qual 공유). 핵심 확정: ① **레짐 대응은 작업 유형(소정비/대PM)이 아니라 Qual 판정(조용/요란)에 건다** — 대PM의 ~70%는 조용이므로 유형 기반 트리거 금지. 피처 `is_post_pm → is_post_loud_pm`(요란 판정 이후), `pm_log.json`에 요란 판정 PM만 기록. ② **처리율 231 wafer/일 정정** (구 170은 train 단독 과소집계 → 3분할 합산 15,919÷69=230.7 실측; label_delay 절대환산 10,200→13,900). ③ **Qual σ-갭 판정식** — 건강 센서 8종(C11·C15·C16·C17·C31·C61·C62·C63, C12 제외) 3σ, 기준=직전 사이클 전체 산포. 운영 주력 = σ-갭 + AE. ④ **시간피처(hour) 실측 확정** — 날짜·배치 통제 후 잔존(진폭 41), 시뮬 재현(합성 Y 생성 시 적용). ⑤ **재분할은 시간(레짐) 기준** — 랜덤 재섞기는 시간피처 성능 착시. ⑥ ~~C25 within-급등 신호(+0.459) 보류~~ → **기각 확정 (2026-07-10, A)**: 레짐 통제 전 confound(레짐 프록시) — M2 gain 23위·직접 투입 CV +3.31 악화, 시뮬 C25-Y 결합 불요. ⑦ **라벨-프리 레벨 프라이어(신규 결정 9)** — 요란 후 예측 천장 문제를 D+0 C17 프라이어(추정)→D+60 rolling bias(실측)→재학습의 3단 릴레이로 해소, 커버리지 486→64 실측(인샘플). ⑧ **레짐 전환 Incident 명시 개설(신규 결정 10/P2)** + **헌법 1-4 "동시 pending 1건" 해석(P3)**. ⑨ 레시피 R2R = 크기를 찾아가는 루프(교락은 split-run 실험 원장으로 점진 해소, F16). **🔒 2026-07-08 사이클 세계관 기획 마감** — 이후 변경은 Incident 방식 재오픈만, 보류 항목은 `docs/사이클_정의_v1.md` §5-1.
> **확정 추가 (2026-07-10~11 — 빌드 단계 확정, 세계관 불변)**: ① **모델·인프라 확정** — 임베딩 bge-m3 hybrid·생성 qwen3:30b-a3b·g5.xlarge 단일 (C 벤치, dual/g6e 불필요). ② **레시피 손잡이 확정** — 파워=C1·가스=C4/C5·시간=C41만(C12·C17·C31 금지), temp·pressure setpoint는 데이터 수집 밖 → 에스컬(temp는 목표 지정형 BL8, W5). ③ **C3478 멘토 확정**(CT①=Cycle 종료·3 cycle / Model R2R clamp ±1×RMSE·요란후 rolling500). ④ **결정 A·B 조합②**(KEEP* 6그룹 분위수 — 헌법 1-1 예외 2, 멘토 확인). ⑤ **DB v4.6** — control_limits 신설(16테이블). ⑥ **C25 기각**(레짐 프록시). 상세는 `일정_v4_task단위.md`·`config_파라미터_합의안_v1.md`·`회의_브리핑_20260711.md`.
> **v4.6-a 멘토 확인 반영 (2026-07-09)**: ⓐ **Recipe R2R 손잡이 = 가변 para(temp·gas유량·pressure, temp 1순위)** 멘토 확정 — SHAP가 지목한 축을 이 가변 para로 변환하는 Process KB 매핑이 옵션① Recipe Tuning의 근거 계층 (temp 가변 시 EPD·장비 상태 para 연동 변화 = 교락의 정상 원인, F16). ⓑ **조치 성공 지표 의미 = "1회 해결"이 아니라 "불량률 개선"** (멘토: "한 번에 해결 불가·구조불량 상존") — 합성 사례 `is_success` 재정의, R2R 튜닝 루프(소폭 조정→검증→재조정)와 정합. ⓒ 정기 리캘리(무이상 시 recal)·BM 사고성은 기존 A1·A4와 정합 — 변경 없음.

---

> **"FDC가 감지하고, SPC/TTTM이 분석하며, AI Agent가 Recipe·실력치·정비 조치안을 근거와 함께 제안하고, 엔지니어의 승인으로 루프가 닫힌다."**

---

## 목차

1. [I. 프로젝트 개요](#i-프로젝트-개요)
   - 프로젝트 주제명
   - 프로젝트 수행 배경 (3대 Problem)
   - 프로젝트 목표 및 수행 내용 (5대 Pipeline)
   - 데이터 환경 및 요구 사항
   - 활용 데이터 형태
2. [II. 상세 구현](#ii-상세-구현)
   - 전체 아키텍처 (7-Layer Stack)
   - Equipment Abstraction Layer — 다중 장비 어댑터
   - Smart SPC 엔진 상세
   - FDC 실력치 보정 Closed Loop 상세
   - RAG 기반 Manufacturing Knowledge Layer
   - AI Agent Decision Support Layer
   - JSON Event Contract v4
3. [III. Scope · 로드맵 · 정리](#iii-scope--로드맵--정리)
   - 서비스 설계 원칙 — "작은 인프라, 완전한 서비스"
   - 구현 범위 (Must / Should → 즉시 구현 / Could → 확장)
   - 8주 실행 계획
   - 기대 효과 및 비즈니스 임팩트
   - Success Metrics
   - Risk & Assumptions
   - 오픈 이슈 & 운영 정책
   - Competitive Differentiation

---

# I. 프로젝트 개요

## 1. 프로젝트 주제명

**Etch 공정 이상 대응을 위한 RAG 기반 제조 AI Agent 플랫폼**

> **Agentic Manufacturing Copilot for Etch FDC-SPC-Limit Control Operations**

반도체 Etch 공정의 FDC Trace 데이터를 기반으로 C65 불량을 예측하고, SPC/TTTM으로 이상 원인을 분석하며, AI Agent가 **Recipe Tuning(R2R)·실력치 재설정·정비 조치안**을 산출해 엔지니어 승인 후 적용한다. 이상 판정 wafer는 **HOLD → 스크랩/해제 Disposition**으로 처리하고, 모델은 PM Cycle 내 **Residual Bias 보정(Model R2R)**과 Cycle 단위 재학습으로 스스로를 갱신한다. 이후 RAG 기반 AI Agent가 과거 유사 사례·공정 매뉴얼·실력치 변경 이력을 검색하여 엔지니어 승인용 근거 리포트를 생성하는 **Human-in-the-Loop 제조 AI 플랫폼**을 구축한다.

v3가 "감지하고, 분석하고, 제어하는" ML 파이프라인 중심이었다면, v4는 **"이상 이벤트를 해석하고, 근거를 수집하고, 조치안을 작성하고, 엔지니어 승인까지 연결하는 지능형 운영 계층"**을 추가한 완전한 제조 AI 플랫폼을 목표로 한다.

---

## 2. 프로젝트 수행 배경 — 3대 구조적 Problem

### PROBLEM 01 · ATTACK 지속 — Inline 공정 중 불량 Attack 누적

Wafer Inline 공정은 통상 **24주(약 6개월)** 가 소요된다 (Flexciton). Etch 공정에서 CD(Critical Dimension) 불량이나 Profile 이상이 발생하더라도 Fab-Out(Wafer Test)까지 추가로 약 **2개월**이 경과되며, 이 기간 동안 원인 Chamber가 방치되어 동일 불량이 연쇄 확산된다. KLA-Tencor의 분석에 따르면, **25k WSPM급 Fab에서 미감지 Excursion을 인라인(<1일)이 아닌 end-of-line(+30일)에서 잡으면 건당 약 $21M(발생당)의 손실**이 발생하며, 소량·장기 excursion에서는 수천~수만 장이 노출되는 일이 드물지 않다 (KLA-Tencor, 2014). 특히 3nm 이하 선단 공정에서는 파운드리 Wafer 단가가 **약 $20,000**(2nm은 $30,000+) 수준으로, 불량 Excursion의 재무적 리스크가 기하급수적으로 커지고 있다 (TrendForce, 2025.09 — 애널리스트 추정).

> "가장 비싼 결함은 인라인에서 잡히지 않은 결함이다(The most expensive defect is the one that wasn't detected inline)." — KLA-Tencor, 2014

Etch 공정은 특히 위험하다. Plasma로 패턴을 한 번 잘못 깎으면, 그 Wafer는 되돌릴 수 없다. 그리고 그 사실을 아는 건 수 주에서 수 개월 뒤다. 특히 Chamber가 다수(6대 이상)로 늘어나면, 동일 Recipe로 운영하더라도 Chamber 간 편차가 불량의 또 다른 축이 된다.

### PROBLEM 02 · DATA DRIFT — PM 이후 예측 모델 성능 열화

Fab-Out 이전 장비에서 초 단위로 측정되는 FDC Trace로 불량을 예측할 수 있게 되었으나, Chamber 예방정비(PM, Chamber Cleaning + Part 교체) 이후 센서 분포가 틀어지며 모델 성능이 급격히 열화되는 치명적 문제가 발생한다. Etch의 경우 Chamber Wall 상태, Focus Ring 마모, Showerhead 막힘 등이 PM 전후로 RF Impedance, DC Bias, Gas Flow 패턴을 크게 변화시킨다. PM 주기는 통상 3~4개월로, 20대 Chamber 기준 연간 수십 회의 수동 재학습이 필요하다. 학술적으로도 **drift(마모에 의한 점진 열화)와 shift(정비에 의한 급변)** 는 구분되며(MDPI *Applied Sciences*, 2025), FDC 전문가들은 "FDC는 엔지니어가 상시 재학습·감시해야 하는 프로세스이며, 임계값이 빡빡하면 알람 폭주·느슨하면 미탐"이라 지적한다(Semiconductor Engineering, 2025.10). 즉 PM 후 재학습은 예외가 아니라 **상시 과제**다.

**현업 확인** *(멘토, 2026-07-06)*: "PM 이후 열화가 안 되는 상황은 없다. 다만 그 열화가 WT(Wafer Test)에 드러나는 경우도, 드러나지 않는 경우도 있다." — 최종 검사만으로는 열화를 다 잡지 못하므로 FDC 기반 조기 감지가 필요하다는 본 프로젝트의 전제를 현업이 확인. 또한 현업은 엔지니어가 decode한 일부 센서만 SPC에 올리며 PM 초기·말기에 문제가 집중되는데, 불량이 난 센서가 SPC 감시 목록에 없었을 수도 있다 — 본 시스템은 **전 센서 모니터링**으로 이 사각을 제거한다.

### PROBLEM 03 · EQUIPMENT SILOS — Chamber 간 단절과 공정 수평 확장의 부재

현재 FDC 시스템은 Chamber별로 독립 운영된다. 단일 Etch Chamber의 센서 데이터는 그 Chamber 안에서만 의미를 가질 뿐, 같은 Equipment(설비) 내 다른 Chamber와의 편차를 추적할 수 없다. TTTM(Tool-to-Tool Matching)은 챔버 간 차이를 정량화하는 표준 방법론으로, 특정 기준 챔버(golden reference)를 지정하는 대신 **동일 공정 챔버 전체 분포(fleet median) 대비 편차**로 판정하는 것이 최신 방향이다 — 불일치 장비일수록 데이터 분산·모드 수가 커진다는 가설 하에 최적 단변량 기법이 분산과 0.95+ 상관을 달성했고(arXiv:2507.10564, 2025.07), AI 챔버 매칭으로 최저 수율 챔버를 0.50→0.81로 끌어올린 실증도 있다(ASMC 2024). 또한 이상 발생 시 과거 유사 사례 검색이나 공정 매뉴얼 참조가 수동으로 이루어지며, 엔지니어의 경험과 기억에 의존한다. Chamber 간 편차를 fleet 수준에서 통계적으로 모니터링하고, **과거 지식을 자동으로 검색·활용하는 지능형 운영 플랫폼**이 필요하다.

| Problem | 핵심 질문 | v3 대응 | v4 대응 |
|---------|-----------|---------|---------|
| **P01** Attack | "Etch 불량을 얼마나 빨리 알 수 있는가?" | Pipeline A + SPC 조기 경보 | Pipeline A + **SPC 조기 경보** + **Agent 기반 즉시 대응 Brief** + **이상 wafer 스크랩 Disposition** |
| **P02** Drift | "Chamber PM 이후 모델이 무너지면 어떻게 하는가?" | Pipeline B + SPC 패턴 감지 + APC 자동 보정 | Pipeline B + **SPC 패턴 감지** + **FDC 실력치 보정 제안** + **RAG 과거 PM 사례 검색** |
| **P03** Silos | "Chamber A의 이상이 Chamber B에 어떤 영향을 주는가?" | Pipeline C (Smart SPC) + Pipeline D (보정 Controller) | Pipeline C + D + **Pipeline E (AI Agent + RAG 통합 의사결정 지원)** |

---

## 2-1. 차별점 — 왜 "능동형(Active)"인가

**기존 솔루션은 "감지"까지고, AITCH는 "판정 → 조치 제안 → 승인 → 적용 → 효과 검증"까지 닫는다.** 이것이 프로젝트명 "**능동형**"의 실체다. 개별 기능은 대부분 이미 상용화되어 있음을 우리는 정직하게 인정한다 — 우리의 차별점은 그것들을 **단일 닫힌 루프로 통합하고, Human-in-the-Loop 승인 게이트로 감사 추적을 확보한 것**이다.

| 기능 | 이미 상용화된 사례 (실증) | 성격 |
|---|---|---|
| C65 예측 (Proxy Metrology) | 삼성 SAIT — 300mm 팹 114,774장 Etch CD 가상계측 (Springer, 2025) | 감지(수동) |
| SPC · TTTM 챔버 매칭 | SandBox Semiconductor 수율 0.50→0.81 (ASMC 2024) · arXiv 분산 기반 TTTM 0.95+ (2025) | 감지(수동) |
| RAG 지식 검색 | Applied SmartFactory · SK하이닉스 Gaia — 부서·업무별 에이전트 (2025~2026) | 감지(수동) |
| **조치 제안 + 승인 + 효과검증 닫힌 루프** | **← AITCH의 통합 지점** | **능동(Active)** |

**"감지"와 "능동"의 차이**: 감지는 "이상이 있습니다"에서 멈춘다. Nelson 위반·TTTM Gap·SHAP 기여도는 전부 "**뭔가 잘못됐다**"는 신호일 뿐, "**무엇을 어떻게 조치할지**"는 알려주지 않는다 — 그 판단은 여전히 사람의 경험·인맥·기억에 의존한다. AITCH는 여기서 한 걸음 더 나아가, 하나의 이상 신호를 **Supervisor Agent가 4지선다로 판정**한다: ① 진짜 장비 이상(→ 정비 + 스크랩), ② 공정 조건 이탈(→ Recipe R2R 튜닝안), ③ 관리 기준선 노후(→ 실력치 재설정), ④ 모두 아님(→ 공정 검토 에스컬레이션). 각 조치안은 **SHAP·유사 사례·매뉴얼 근거(Evidence Card)와 함께 제안**되고, **엔지니어 승인 후에만 적용**되며, 적용 결과가 **효과 검증**되어 다시 학습 자산(Approval Log)이 된다.

**"AI가 조치를 제안한다"만으로도 차별화가 아니다 — 핵심은 승인 루프다.** 조치 제안 자체는 일부 상용 플랫폼도 시도한다. AITCH의 진짜 차별점은 **무승인 자동 적용을 금지하되, 제안 → 승인 → 적용 → 효과 검증 → 승인 이력 학습까지 하나로 닫고, HITL 승인 게이트를 아키텍처의 first-class 요소로 두어 전량 감사 추적을 확보**하는 데 있다. 현업 R2R은 레거시 DB API로 자동 실행되지만 설명·감사 추적이 없다. 이는 최근 업계 모범사례로 정착 중인 방향과 정확히 일치한다 — "레시피·장비 파라미터·품질 홀드를 바꾸는 모든 에이전트 행위는 사람 인가 필수 + 전량 로깅(bounded autonomy)"(Athena, 2026), 그리고 "Critic 에이전트가 후보안을 물리적 타당성·안전 한계로 판정해 {accept·revise·escalate}로 중재하고, 판단 불가 시 구조화 리포트를 사람 승인으로 넘긴다"(MAKA, arXiv 2026).

> **한 문장 요약**: *"기존은 감지까지, AITCH는 근거 있는 조치 제안과 그 검증까지 — 사람이 승인하는 능동형 닫힌 루프."*
>
> 상세 근거(페르소나 4종·Pain Point 10축·실증 출처 31건)는 `페르소나_반도체_공정_엔지니어_v4.6.md` 참조.

---

## 3. 프로젝트 목표 및 수행 내용 — 5대 Pipeline

실제 반도체 Etch 공정의 Sensor Data를 분석하여, **예측(A) → 감지(B) → 분석(C: SPC/TTTM) → 판단(E: RAG+Agent) → 보정·조치(D: R2R + Limit Control + Disposition)** 의 5대 Pipeline과, 이를 지속적으로 갱신하는 **모델 갱신 루프 3종**(⓪ Model R2R — Cycle 내 Residual Bias 자동 보정 / ① LightGBM — Cycle 종료 시 sliding window 자동 재학습 / ② Autoencoder — PM 후 Qual 검증 + 엔지니어 승인 기반)으로 구성된 Human-in-the-Loop Closed Loop를 구현한다. *(v4.5: 멘토 제안 시퀀스 반영)*

### Pipeline A · Wafer 단위 — 불량 칩 개수(C65) 정밀 예측

- WF(C64) 단위로 Etch 챔버에서 나온 FDC Trace data를 집약한 Feature 기반 회귀
- 공정 Step(C7)별 통계·추세 파생 변수 설계 — 특히 RF Power/Reflect, DC Bias, Chamber Pressure 등 Output Parameter의 Step 내 안정성 변화 추적
- **LightGBM + SHAP** 기반 메인 예측 모델 (설명 가능성 우선)
- SHAP 기반 Top-3 불량 기여 인자 산출 (**모델 원본 순위 유지**) — Nelson Rule 위반 센서는 **별도 태그(SPC Flag)로 병렬 제시**. 룰 기반으로 SHAP 순위에 개입하면 설명 가능성이 훼손되므로, 모델 근거(SHAP)와 통계 근거(Nelson)를 독립 채널로 비교 가능하게 함 *(v4.4)*
- **Model R2R** *(v4.5 신설 — 멘토 제안)*: PM Cycle 초반 실측 C65 도착 시 Residual(실측-예측)을 추적, 급격한 격차 발생 시 **Residual만큼 Bias를 모델 출력에 자동 update**. 물리 세계를 건드리지 않으므로 승인 불요 — 단 ⓐ 보정폭 상한(config, train RMSE 기준) 내에서만 자동, 초과분은 clamp 후 **Incident 신호로 전환**("모델이 낡은 게 아니라 챔버가 평소의 PM 후와 다르게 거동 중"), ⓑ 매 갱신을 `ct_decisions`에 기록하고 `fdc.prediction`에 bias 버전 표기
- **평가지표**: RMSE(성과) + **R²(현업 중시 — v4.5 추가)**. ⚠️ 현 모델 R²가 비정상적으로 높음 → 누수(헌법 1-3) 여부 최우선 검증

### Pipeline B · Chamber 단위 — Chamber 건전성·이상 감지

- Chamber 단위(C24 — train은 단일 챔버, 다중 챔버는 시뮬레이터 생성) 시계열 기반 건전성 추적 — Vdc(C11) 추세, Gas Flow Consistency, 열 이력(C17/C9)
- PM(Chamber Cleaning + Part 교체) 전·후 분포 변화로 Drift 조기 탐지
- **Autoencoder(PyTorch) 확정** — Reconstruction Error를 Anomaly Score로 산출 및 경고. 통계 기반 Outlier 탐지는 Could로 이동 *(v4.4: 핵심 방법론 미확정("또는") 제거)*
- **TTTM(Tool-to-Tool Matching)**: 동일 공정 챔버 전체 분포(fleet median) 대비 센서 Scale 차이 계산 (지정 기준 챔버 없음 — 멘토 확인)

### Pipeline C · Smart SPC Engine

- **Nelson Rules 자동 판정** *(v4.4: "감지"→"판정")*: N1~N8 전체를 모든 Chamber 센서에 대해 실시간 평가하고 위반 패턴을 **`spc_violations`에 기록** (1차 적용: 센서 실측값). C65 예측값에는 **보조 모니터링**으로 별도 적용 (모델 출력 추세 추적 목적)
- **TTTM 비교 판정**: 동일 설비(Equipment) 내 Chamber 간 편차를 산출하고 **`tttm_comparisons`에 기록**
- **Hybrid Architecture**: Tier 1 — Classical SPC (Nelson Rules, Audit Trail) / Tier 2 — AI/ML (Autoencoder sub-sigma drift)
- ***C의 경계*** *(v4.4)*: C는 **"N몇이 어떤 센서에서 걸렸다"는 사실 기록**까지. "왜 걸렸는가(예: Focus Ring 마모)"의 원인 추론은 Pipeline E가 Process Knowledge DB와 결합하여 수행

### Pipeline D · R2R & 보정 Controller (Recipe·Limit·Disposition) *(v4.5 재개정 — Recipe R2R 부활)*

- **Recipe R2R** *(v4.5 부활 — 멘토 정정: "레시피 수정도 서비스에 넣어야 한다")*: 모델의 **SHAP 분석 + Process Knowledge RAG**로 Recipe Tuning 안(파라미터·방향·크기) 산출 → 엔지니어 승인 → **시뮬레이터의 해당 챔버 setpoint·분포에 적용** → 적용 후 N wafer 효과 검증. 현업은 Agent가 레거시 DB API로 R2R을 자동 진행하나 본 시스템은 승인 게이트 유지(차별점). 무승인 자동 적용은 여전히 금지 (헌법 1-1)

- **실력치(Baseline) 보정**: Pipeline B의 Drift 감지 → Pipeline C의 SPC 분석 → 분포 이동이 "정상 상태 변화(PM 후 등)"로 판정되면 **rolling mean±3σ 기반 실력치(Control Limit) 재산정안** 산출 (멘토 확인 산식) **신호-조치 경계** *(v4.5)*: 실력치는 SPC 통계로만 산출 — 모델 신호(SHAP·예측값)로 실력치를 움직이지 않는다 (멘토: "실력치 R2R은 Model 예측값으로 하긴 어렵다"). 모델 신호는 Recipe Tuning 제안으로 라우팅.
- **C65 Proxy Metrology** *(v4.4: "Virtual Metrology"에서 명명 변경)*: 현 데이터셋에 Post-Etch CD 실측 라벨이 없으므로, Pipeline A의 C65 예측값을 공정 품질 Proxy로 활용 → **wafer HOLD/스크랩 Disposition 판정 근거**. 향후 CD 실측 확보 시 본격 VM으로 확장
- **Interlock & Disposition**: 인터락 발생 시 해당 wafer/LOT **HOLD** → 엔지니어 승인 후 **SCRAP(폐기) 또는 RELEASE(해제)** — Etch 이상 wafer는 rework 불가하므로 스크랩이 원칙
- **Human-in-the-Loop Gate**: Recipe 변경·실력치 변경·정비 조치·스크랩 판정은 AI Agent가 권고 → 엔지니어 승인 → 실행
- 실력치 보정 시뮬레이션 — 재설정안 적용 시 가성알람 감소율·미탐 위험 사전 추정

> **v4.3→v4.5 경위**: v4.3에서 멘토 피드백을 "Recipe 보정 전면 금지"로 해석했으나, 이후 멘토가 정정 — 금지 취지는 **무승인 자동 변경**이었고, SHAP 기반 Recipe Tuning을 승인 게이트를 거쳐 적용하는 R2R은 오히려 서비스에 포함되어야 한다("넣어야 한다"). 이에 조치 옵션 4종 확정: ① Recipe R2R(공정 조건 이탈), ② 실력치 재설정(기준선 노후), ③ 정비+스크랩(진성 열화), ④ 에스컬레이션.
>
> **wafer 라우팅 룰** *(v4.5 — 멘토: "한 장만 높게 나오면 스크랩, 여러 장이면 R2R")*: crazy wafer 1장(spot) = 자동 HOLD → 승인 스크랩 / 여러 장 연속 = R2R 경로(Model R2R + Recipe 제안 검토) / 연속·심각 = RTD(inhibit) + BM 정비. PM 직후 대량 이상 = 장비 진행 금지 + 대부분 폐기(현업 확인).

### Pipeline E · RAG 기반 AI Agent Decision Support *(v4 신규)*

- **원인 추론** *(v4.4: E의 존재 이유 명확화)*: Pipeline C가 기록한 위반 사실(`spc_violations`, `tttm_comparisons`)에 **Process Knowledge DB**(Etch 공정 원리, 센서 의미, 파라미터 영향)를 RAG로 결합하여 원인 후보를 추론 (예: "DC Bias에서 N3 위반" + Process Knowledge → "Focus Ring 마모 가능성")
- **fast-path / slow-path 분리** *(v4.4)*: 심각도 평가·자동 동작(inhibit/HOLD)은 **규칙 기반 동기 경로(sub-second, LLM 미사용)**, RAG 검색·원인 추론·Brief 생성은 **비동기 경로(RAG+LLM)** — 알람 대응 속도와 분석 깊이를 분리
- **옵션 리포트 3종 병렬 생성** *(v4.5)*: ① Recipe Tuning(**delta 수치는 B의 튜닝 엔진 API**, C는 Process KB 근거+서술), ② 실력치 재설정(수치는 B 엔진 API), ③ 정비(Error Manual RAG) → Supervisor가 4지선다 판정 + 순위. **정량은 B·근거는 C** 패턴을 옵션 전반에 일관 적용
- **RAG 기반 근거 검색**: **지식베이스 5타입**(Error Manual / Process Knowledge / Historical Case / Limit Change Log / **Recipe R2R 이력** — 2026-07-07 C 제안 채택)에서 유사 사례·이력 자동 검색. 물리 저장은 Qdrant 벡터 3컬렉션 + PostgreSQL 이력 테이블 (5-1절)
- **인제스트 파이프라인** *(v4.5 — 멘토 조언 / 2026-07-06 C 검토 정정)*: PDF/PPT는 **페이지(장) 단위 청킹 → Vision LLM 해석 → Qdrant 적재**. 임베딩 모델 = **bge-m3 (dense+sparse 하이브리드) ✅확정 (C 벤치 2026-07-09 — 골든셋 47문항 MRR 0.901 1위, Qwen3-Embedding 0.887·KORPatent 0.808 탈락)**. 생성 LLM = **qwen3:30b-a3b ✅확정 (재대결 7/10)**, 운영 = **g5.xlarge 단일** (config D7·D9·D20). 표기: 공정명·step명 영어, 설명 한글
- **KB 시딩 규모** *(✅달성 2026-07-11)*: **KB 5타입 합산 13,481청크** (목표 1만 3주 선행 — 벡터 3컬렉션 error_manual 586·process_knowledge 106·historical_case 10,000 + PG 로그 limit 1,709·recipe 1,080). Historical Case는 실제 조치내역 확보 불가 → 시뮬레이터 시나리오 기반 합성(P4-5 v2 — is_success='개선', 손잡이 C1/C4/C5, D6 상한). near-duplicate 대량 생성 금지 · provenance 라벨 유지
- **Evidence Card 기반 근거 제시**: 검색 결과를 구조화된 형태로 엔지니어에게 제공
- **승인용 Brief 자동 생성**: 원인 후보, 과거 사례, 위험도, 추천 조치를 요약한 Approval Report 생성
- **CT 재학습 필요성 판단 보조**: Drift, 검증 성능, 불량 데이터 기반 재학습 Trigger 제안

### 전체 Human-in-the-Loop Closed Loop 흐름

```
Etch FDC 센서 데이터
   ├──(병렬)──→ Pipeline A (C65 예측 + SHAP) ─── 근거 컨텍스트 ──┐
   └──(병렬)──→ Pipeline B (Drift/Outlier 감지 + TTTM Scale 산출) │
                     ↓                                            │
        Pipeline C (Nelson Rules 판정 + TTTM 비교 → DB 기록)       │
                     ↓                                            │
        E · fast-path (규칙 기반 심각도 평가, 동기, sub-second) ◄──┘
          ├─ 자동 동작: 의심 윈도우 계산 → wafer HOLD, Critical 시 chamber inhibit
          └─ Incident 그룹핑 (승인 요청은 사건당 1건)
                     ↓ (이진 분기가 아님 — 리포트 3종을 항상 병렬 탐색: Recipe·실력치·정비)
        ┌────────────────────────────────────────────────────┐
        │ 옵션①: Recipe R2R — SHAP+Process KB 튜닝안 (v4.5)  │
        │ 옵션②: 실력치 재산정(mean±3σ) + Limit Simulation   │
        │ 옵션③: 정비 조치안 — Error Manual RAG 검색         │
        └────────────────────────────────────────────────────┘
                     ↓
        E · slow-path (RAG+LLM, 비동기)
          ├─ 과거 유사 Case 검색 (Historical Case DB)
          ├─ Limit Change Log 검색
          ├─ Process Knowledge DB로 원인 추론
          ├─ 옵션별 Evidence Card 생성 + wafer Disposition 권고
          └─ Supervisor: 4지선다 판정 + 세 옵션 순위 → Approval Brief
                     ↓
        Approval Gate (LangGraph interrupt = Pending)
          → 엔지니어가 최종 선택 (승인/수정/반려/에스컬레이션)
                     ↓
   조치 실행: 실력치 반영(fdc.correction → SPC 한계 갱신) / 정비 / 스크랩·해제
                     ↓
   조치 내용 저장 (approval_records) → RAG 학습 자산으로 축적
                     ↓
   CT 재학습 (2종)
     ├─ ① LightGBM: 실측 Y 지연 피드백 → 자동 재학습 (champion-challenger 통과 시 배포) → A 갱신
     └─ ② Autoencoder: PM 후 Qual Wafer 검증 + 엔지니어 승인 → 재학습 → B 갱신
```

---

## 4. 데이터 분석을 위한 환경 및 요구 사항

### Data Engineering

- **Kafka** 스트리밍 모사 — 정적 CSV를 multi-topic으로 발행
- **Airflow** 스케줄링 — 전처리, 학습, SPC 배치, APC 피드백 루프 오케스트레이션
- **PostgreSQL / TimescaleDB** — 시계열 SPC 데이터 마트, Wafer/Chamber/SPC/APC/Agent Mart

### ML & MLOps

- **LightGBM** — Pipeline A 예측 모델 (SHAP 설명성 우수)
- **Autoencoder (PyTorch)** — Pipeline B 이상 탐지, Pipeline C sub-sigma drift
- **Optuna** — 하이퍼파라미터 최적화
- **MLflow** — 실험 추적, 모델 버전 관리

### Smart SPC

- **Nelson Rules Engine** — Python Custom 구현 (N1~N8 전체)
- **TTTM 비교 분석** — 동일 공정 챔버 분포(fleet median) 대비 센서 Scale 차이 계산

### 실력치 보정 Controller

- **실력치 재산정 엔진** — rolling mean±3σ 리캘리브레이션 (정기 자동 + 이상 대응 승인 2트랙)
- **Limit Simulation** — 실력치 재설정 시 가성알람 감소율·미탐 위험 추정

### RAG & AI Agent *(v4 신규)*

- **Qdrant** — Vector DB (유사 Case/Manual 임베딩 검색 + **payload 필터**: equipment/chamber/sensor/rule_id/PM 근접도 조건과 벡터 유사도를 한 쿼리로 처리. FAISS는 메타데이터 필터 부재로 교체 — v4.3)
- **LangGraph** — Tool-Using Agent 프레임워크 + 승인 상태머신 (interrupt + Postgres 체크포인터)
- **FastAPI** — Agent Tool API 서빙

### Product Engineering

- **FastAPI** — 추론 + SPC + APC + Agent 통합 API 서빙
- **Docker Compose** — E2E Demo 환경
- **React** — 통합 대시보드 (FDC Predict + SPC Chart + Nelson Rules Viewer + 승인 큐/Approval Brief + Agent Report). FastAPI 백엔드와 REST + WebSocket/SSE(실시간 알림) 연동 — 상세: `docs/프론트_화면_정의서_v1.md` *(v4.4: Streamlit → React 전환)*

---

## 5. 활용되는 데이터 형태

### Input · 원천 데이터

- SK하이닉스 현장 CSV — train_data.csv (11,939 WF / 123,614 Row), valid_X / test_X (각 1,990 WF)
- Etch 공정 FDC Trace: 1차원 시계열 센서 데이터, 3초 간격 샘플 — 주요 컬럼(공식 정의): **C64** Wafer ID, **C20** Lot ID(+C34 슬롯), **C6** Recipe ID(2종), **C7** Step(5개, Step 4가 메인 플라즈마), **C24** Chamber(단일), **C33** RF time(RF 누적 시간 — 리셋=PM, 멘토 확인), **C65** Target. 전체 컬럼 명세: `docs/데이터_사전_v1.md` *(v4.4 정정: 기존 "C6=Chamber, C14=Equipment, C33=PM Flag" 표기는 오류)*
- ⚠️ **train 데이터는 Chamber 1대·Recipe 2종** — 다중 Chamber(TTTM)는 시뮬레이터가 생성한다

### Output · Data Mart 구성

| Mart | Pipeline | 내용 |
|------|----------|------|
| **Wafer Mart** | A | C64 기준 예측 모델 학습용 정제 데이터, SHAP 원인 분석 결과 포함 |
| **Chamber Mart** | B | Chamber(C24) 기준 시계열 열화·PM 추적, Drift Score, TTTM 비교 결과 |
| **SPC Mart** | C | Nelson Rule 평가 결과, TTTM 비교 결과, SPC Alert 이력 |
| **Correction Log** | D | 실력치 변경 이력, 보정 파라미터, 승인자, 가성알람 감소 실적, wafer Disposition 이력 |
| **Agent Mart** *(v4 신규)* | E | Agent 분석 리포트, RAG Evidence Cards, 승인/반려 이력, CT 권고 이력 |

---

# II. 상세 구현

## 1. 전체 아키텍처 — 7-Layer Stack *(v4.4 개정)*

> **v4.4 변경**: ① E를 L4(fast-path·동기·규칙)와 L6(slow-path·비동기·RAG+LLM)으로 물리 분리, ② 저장소를 모든 Layer가 공유하는 수직 영역으로 분리, ③ Presentation 계층 신설, ④ Layer 번호와 실행 순서 일치.

```
        (처리 흐름: L1 → L7, 데이터는 위에서 아래로)

┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 · DATA & EVENT                                          │
│   Kafka Event Bus (6 Topic) · FDC Trace Producer (CSV) · EAL     │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2 · DETECT & PREDICT                          [병렬 실행] │
│   A · C65 예측 (LightGBM+SHAP, per-wafer)                        │
│   B · Outlier/Drift (Autoencoder) + TTTM Scale (per-chamber)     │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3 · ANALYZE (Smart SPC)                                   │
│   Nelson Rules 판정(N1~N8) · TTTM 비교 → spc_violations 기록      │
│   (Tier1 Classical + Tier2 AI/ML Hybrid)                         │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4 · SEVERITY ASSESSOR — E fast-path (규칙·동기·LLM 미사용) │
│   하드알람/인터락 판별 · 실력치 재설정 폭 상한 체크 · SHAP/TTTM   │
│   위반 센서 판별 → 심각도 평가 (sub-second)                       │
│   자동 동작: 의심 윈도우 → wafer HOLD · Critical 시 inhibit       │
│   Incident 그룹핑 → 리포트 3종 병렬 탐색 개시 (Recipe·실력치·정비) │
└──────────┬───────────────────────────────────┬──────────────────┘
           ▼ (동기)                             ▼ (비동기)
┌──────────────────────────────┐  ┌───────────────────────────────┐
│ LAYER 5 · CORRECTION         │  │ LAYER 6 · DECISION SUPPORT    │
│ (Limit Control) [옵션②]      │  │ — E slow-path (RAG+LLM)       │
│  실력치 재산정 (mean±3σ)     │─▶│  Root Cause (Process KB)      │
│  Limit Simulation            │  │  옵션③ 정비안 (Error Manual)  │
│  C65 Proxy Metrology         │  │  유사 Case·Limit Log 검색     │
│  (→ Disposition 근거)        │  │  Evidence Card · CT Advisor   │
└──────────────────────────────┘  │        ▼                      │
                                  │  Supervisor (4지선다 + 순위)  │
                                  │        ▼                      │
                                  │  Approval Gate                │
                                  │  (LangGraph interrupt)        │
                                  └──────────────┬────────────────┘
                                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 7 · PRESENTATION (React SPA + FastAPI)                    │
│   SPC/TTTM Dashboard · 승인 큐(우선순위·claim) · Approval/Brief UI │
│   Agent 추천 KPI (채택률/수정률/반려율) · 자연어 질의             │
│   → 엔지니어 최종 선택: 승인/수정/반려/에스컬레이션               │
└─────────────────────────────────────────────────────────────────┘

  ※ STORAGE (모든 Layer 공유, 수직 영역):
     TimescaleDB(FDC 시계열) · PostgreSQL(SPC/보정/승인/Incident)
     · Qdrant(매뉴얼·과거 Case) · Data Mart 5종(Wafer/Chamber/SPC/
     Correction Log/Agent)

  ※ 동기 경로: L1→L2→L3→L4→L5(실력치 산출)→L7 Alert  [M8 < 5초]
  ※ 비동기  : L4 트리거 → L6(RAG+LLM) → Brief → Approval Gate 합류
  ※ 조치 반영: 승인 → fdc.correction 발행 → L3 SPC 한계 갱신(B 구독)
              · 정비 조치 · wafer SCRAP/RELEASE

  ╔═══════ CT RETRAINING LOOP (Layer 외부, 재학습 순환 2종) ═══════╗
  ║ ① LightGBM (자동): 실측 Y 도착 → 채점 → 자동 재학습            ║
  ║    → champion-challenger 통과 시 배포 → L2·A 갱신              ║
  ║    ※ 재학습 트리거 시점 미확정 (실측마다/일배치/월배치 — 오픈) ║
  ║ ② Autoencoder (Qual 기반): PM 완료 → 설비 오프라인             ║
  ║    → Qual Wafer 투입 → A/B/C가 Qual 결과 산출 → Dashboard 제시 ║
  ║    → 엔지니어 승인 → AE 재학습 + 실력치 재산정 → L2·B 갱신     ║
  ╚════════════════════════════════════════════════════════════════╝
```

### Layer 간 데이터 흐름

| From → To | 데이터 | 트리거 | 경로 |
|-----------|--------|--------|------|
| L1 → L2 | 정제된 WF/Chamber Feature | Kafka Consumer (실시간) / Airflow Batch | 동기 |
| L2 → L3 | C65 예측값 + SHAP + Drift/Anomaly Score | 매 Wafer 처리 완료 시 | 동기 |
| L3 → L4 | Nelson Rule 위반 + TTTM Gap + SHAP 근거 | Rule 위반 즉시 | 동기 |
| L4 자동 동작 | 의심 윈도우 → wafer HOLD (LOT 파생) · Critical 시 chamber inhibit | 심각도 평가 즉시 (승인 불요) | 동기 |
| L4 → L5 + L6 (병렬) | 심각도 점수 + Incident Context | 리포트 3종 탐색 동시 개시 | L5 동기 / L6 비동기 |
| L5 → Approval Gate | 옵션② 실력치 재산정값 + Limit Simulation 결과 | 재산정값 산출 시 | 동기 |
| L6 → Approval Gate | 옵션①(Recipe)+②(실력치)+③(정비) Evidence + Disposition 권고 + **순위 추천 (4지선다)** | RAG 검색 + LLM 생성 완료 시 | 비동기 |
| Approval Gate → 엔지니어 (L7) | Approval Brief (Incident당 1건) | 합류 완료 시 | — |
| 엔지니어 → 조치 | 실력치 반영 **또는** 정비 실행 **또는** 에스컬레이션 + wafer SCRAP/RELEASE | **엔지니어 최종 선택** 시 (승인/수정/반려) | — |
| 승인 → L3 *(v4.3)* | 승인된 실력치 (`limit_version`, `effective_from`) — `fdc.correction` 발행, **팀원 B가 구독하여 SPC 한계 즉시 갱신** | CorrectionApplied 시 | 비동기 |
| CT① → L2·A | 재학습된 LightGBM | **PM Cycle 종료** → sliding window(최근 3개월 상당) 자동 재학습 → champion-challenger 통과 시 배포. Cycle 내에는 Model R2R(bias)로 대응 *(v4.5)* | 지연·자동 |
| CT② → L2·B | 재학습된 Autoencoder | PM 후 Qual 검증 통과 + **엔지니어 승인** 시 (실력치 재산정과 공통 트리거) | Qual 완료 후 |
| L3 → L6 *(v4.3)* | Incident Reopen (조치 후 동일 패턴 재발) | verifying 구간 내 재알람 시 | 비동기 |

---

## 2. Equipment Abstraction Layer (EAL) — 다중 Chamber·공정 어댑터

### 설계 원칙

Applied Materials의 AppliedTwin™ Golden Chamber Model과 Siemens의 Digital Thread 접근법에서 영감을 받은 패턴. 각 Chamber/공정 유형의 특수성을 추상화 계층으로 격리하여, 새로운 Chamber가 추가되어도 Pipeline A~E의 핵심 로직을 수정하지 않고 Schema 등록만으로 통합 가능하게 한다.

### Equipment Schema Registry

```
equipment_schema/
├── etch/
│   ├── equipment_01/           # 설비 1 (Chamber 그룹)
│   │   ├── sensors.yaml        # RF Power(Vpp/Vdc), Pressure, Gas Flow, DC Bias
│   │   ├── pm_config.yaml      # PM 주기 4개월, Seasoning 24h
│   │   └── limit_params.yaml   # 실력치 재산정 파라미터 (rolling N, 폭 상한)
│   └── equipment_02/           # 설비 2
│       ├── sensors.yaml        # 동일 센서, RF Impedance Baseline 높음
│       ├── pm_config.yaml      # PM 주기 3개월, Seasoning 48h
│       └── limit_params.yaml   # 실력치 재산정 파라미터 (장비별 override 가능)
├── nelson_config.yaml          # 공통 Nelson Rule 민감도
└── spc_config.yaml             # 설비별 Control Limit 정의
```

### Equipment Adapter Interface

```python
class EtchChamberAdapter(ABC):
    """모든 Etch Chamber가 구현하는 표준 인터페이스"""

    chamber_id: str               # "E14_1", "E14_2", ...
    equipment_type: str = "ETCH"

    # --- Pipeline A/B 용 ---
    @abstractmethod
    def extract_features(self, trace: TraceData) -> WaferFeatures:
        """Raw Etch Trace → 집약 Feature"""
        ...

    @abstractmethod
    def get_sensor_metadata(self) -> List[SensorMeta]:
        """RF Power(Vpp, Vdc), Chamber Pressure, Gas Flow, DC Bias 등"""
        ...

    # --- Pipeline C 용 ---
    @abstractmethod
    def get_spc_config(self) -> SPCConfig:
        """Etch 핵심 SPC: CD Uniformity Cpk, TTTM 기준 등"""
        ...

    # --- Pipeline D 용 ---
    @abstractmethod
    def get_monitored_limits(self) -> List[LimitParam]:
        """실력치(관리 기준선) 관리 대상 센서 파라미터"""
        ...

    @abstractmethod
    def compute_limit_offset(
        self, spc_alert: SPCAlert, drift_context: DriftContext
    ) -> LimitOffset:
        """SPC 위반 + Drift Context → 실력치 재산정값 (rolling mean±3σ)"""
        ...
```

---

## 3. Smart SPC Engine 상세

### 3-1. Nelson Rules Engine — Etch 특화

8개 Nelson Rule(N1~N8)을 Etch Chamber의 모든 센서(Output Parameter)에 대해 실시간 **판정**하고, 위반 사실을 `spc_violations`에 기록한다. C65 예측값에는 보조 모니터링으로 별도 적용한다. *(v4.4: 컬럼을 "관측 패턴(C가 기록하는 사실)"과 "원인 추론 예(E가 Process Knowledge DB로 도출)"로 분리 — C와 E의 역할 경계 반영)*

| Rule | 조건 | 관측 패턴 (C가 기록) | 심각도 | 원인 추론 예 *(E가 Process Knowledge DB로 도출)* |
|------|------|---------------------|--------|-----------------------------------------------|
| **N1** | 1점 ±3σ 초과 | RF Power 급락, Chamber Pressure 급등 | Critical | Arc 발생 가능성, Leak 의심 |
| **N2** | 9점 연속 동일 방향 | Impedance 지속적 변화 | Warning | Chamber Wall 증착 누적 |
| **N3** | 6점 연속 증감 | DC Bias 점진적 상승 | Warning | Focus Ring 마모 |
| **N4** | 14점 연속 증감 반복 | RF Matching Oscillation | Warning | Matching Network 불안정, 온도 제어기 Hunting |
| **N5** | 2/3점 ±2σ 초과 | PM 직후 센서값 편차 | Warning | Seasoning 불충분, Gas MFC Calibration Shift |
| **N6** | 4/5점 ±1σ 초과 | 센서값 미세 이탈 시작 | Info | Chamber Aging 초기 — Showerhead, ESC 열화 |
| **N7** | 15점 ±1σ 이내 | 복수 Chamber 간 정상 범위 차이 | Info | Chamber Mismatch |
| **N8** | 8점 ±1σ 밖 | 두 Population 혼재 | Warning | 양호/불량 Chamber 불균일 |

**구현**: Sliding Window에 대해 Stateless Rule Evaluator. 위반 시 `{chamber_id, sensor_id, rule_id, severity, timestamp, current_value, control_limits}` 형식으로 SPC Mart에 저장하고, 동시에 Pipeline E(AI Agent)에 이벤트 전파.

### 3-2. TTTM (Tool-to-Tool Matching) 비교 분석

동일 공정 챔버 전체 분포(fleet median) 대비 각 Chamber의 센서 평균/분산 Scale 차이를 계산. 불량 wafer 그룹에서 차이가 더 크게 나타나는 패턴을 식별.

```
TTTM Score = Σ |sensor_mean(Chamber_X) - sensor_mean(Reference)| / σ_reference

Threshold: TTTM Score > 2.0 → Warning, > 3.0 → Critical
```

**TTTM 참조 방식** *(멘토 확인 완료 — 2026-07-03)*: 현업은 특정 golden chamber를 지정하지 않고, **동일 공정을 수행하는 챔버·장비 전체(장비 4~5대, 챔버 ~20대)의 FDC Summary 값을 Box Plot 분포로 비교**하며, 챔버별 실력치가 다르므로 공정 엔지니어가 도메인 지식으로 판단한다. 본 시스템은 이를 그대로 구현한다:

- **주 방법**: 동일 공정 챔버 분포 비교 — Box Plot 시각화(S2), 정량화 지표는 fleet median 대비 편차(TTTM Score). *"엔지니어의 도메인 지식 판단"을 Agent가 근거와 함께 보조하는 것이 본 시스템의 역할*
- **보조 방법**: 각 챔버의 **Qual 통과 시점 스냅샷 대비 절대 비교** — 전체 챔버가 함께 drift하는 공통 원인 케이스는 상호 비교(Box Plot)로 잡히지 않으므로, 자기 과거 지문 대비 비교가 유일한 감지 수단 (역방향 룰 4-5절의 근거). Qual 판정 **σ-갭 < 3σ** *(2026-07-11 교체 — 구 "TTTM 갭 < 1%")*도 이 스냅샷 기준
- ~~지정 golden chamber 참조~~ **제거** (멘토 답변 반영 — 현업 방식 아님)

> **참고**: Cp/Cpk(공정능력지수) 산출은 USL/LSL 사양 한계가 데이터에 포함되어 있지 않아 현 데이터셋으로는 한계가 있으므로 MVP 범위에서 제외한다. 향후 실제 Fab 연동 시 Spec Limit이 확보되면 추가 구현 가능하다.

### 3-3. Hybrid Architecture: Classical + AI SPC

```
Tier 1 (Classical SPC)              Tier 2 (AI/ML SPC)
─────────────────────               ─────────────────────
• Nelson Rules N1~N8                • Autoencoder Reconstruction Error
• TTTM Chamber Matching             • Sub-Sigma Drift Detection
• Shewhart Control Charts           • SHAP으로 Rule 위반 원인 설명
• Audit Trail                       • Pre-Rule Alert (AI 선제 감지)
```

---

### 3-4. PM 후 Qualification Wafer 검증 (RECOVERY_VERIFY) *(v4.4 신규)*

PM 이후 설비는 오프라인 상태이며 진행 중 Wafer는 타 설비로 이관된다. PM 완료 후 **소량의 Qualification Wafer(Qual)**를 투입하여 PM 정상 수행 여부를 검증하고, 이 결과가 **Autoencoder 재학습과 실력치 재산정의 공통 트리거**가 된다 (PM 후 "새로운 정상 상태"의 확정 절차).

```
PM 완료 → 설비 오프라인 유지
    ↓
Qual Wafer 소량 투입 (검증용)
    ↓
Pipeline A: Qual C65 예측 (정상 범위 확인)
Pipeline B: Qual Anomaly Score + TTTM Score 산출
Pipeline C: Qual Nelson Rules 판정 (위반 여부 확인)
    ↓
Dashboard에 Qual 결과 요약 제시
    ↓
엔지니어 판단:
  ├─ 통과 → 승인 → ① Qual 데이터를 새 '정상' 기준으로 AE 재학습
  │              → ② 동일 데이터로 실력치 재산정 (v4.3 연계)
  │              → ③ Qual 스냅샷 저장 (TTTM 보조 참조 갱신 — 3-2절)
  │              → 설비 정상 복귀 (ChamberRequalified)
  └─ 미통과 → 추가 Qual 투입 또는 재정비
```

**검증 기준 (Threshold):**

| 검증 항목 | 기준 | 출처 Pipeline | 판정 목적 |
|-----------|------|--------------|----------|
| AE Anomaly Score (Qual 평균) | < 0.2 | B | 기존 AE 기준으로 극단적 이상이 아닌지 확인 |
| **σ-갭** (스냅샷 대비) | **< 3σ** | B | *(2026-07-11 교체 — 구 "TTTM 갭 < 1%")* 기준 산포=직전 사이클 전체·C12 제외·과도 제외 (F12). 운영 주력 = σ-갭+AE |
| Nelson Rule 위반 | 0건 | C(→B) | 통계적 이상 패턴 없음 확인 |
| C65 예측 | 정상 범위 내 | A | ⚠ proxy 예측값(Qual은 WT 미경유) — 과도기 기여 없음 |

> **핵심**: Qual 검증은 "PM이 잘 됐는가"의 확인 절차이고, 통과 시 Qual 데이터가 AE 재학습의 학습 데이터이자 실력치 재산정의 표본이 된다. 재학습·재산정 여부는 **엔지니어가 최종 승인**한다 (Human-in-the-Loop).

**현업 절차와의 대응** *(멘토 추가 확인 — 2026-07-06)*: 현업의 챔버 이상 대응은 "RTD 배차 제외 → BM(사후 정비) → Sample wafer → 이전 실력치로 계측" 순서이며, 본 시스템과 1:1 대응한다. 또한 현업도 TTTM을 상시 점검하고 챔버별 실력치가 달라 관리선 초과 시 조치한다 — 본 설계와 동일.

| 현업 | 본 시스템 |
|---|---|
| RTD(Real-Time Dispatcher) 배차 제외 | 자동 inhibit (Critical 시) |
| BM (Breakdown Maintenance) | Maintenance Agent 권고 → 승인 → 조치 |
| Sample wafer 투입 | Qual wafer 5장 (QUAL 모드) |
| 이전 실력치로 계측 | Qual 4지표를 **기존 기준선 대비** 판정 → 통과 후에야 재산정 |

※ 골든패스(장비 조합 최적화)·dedication·n대화(양산 전개)는 스케줄링/램프업 도메인으로 **범위 외** — 멀티챔버 구성은 "n대화 이후" 상태를 가정.

#### 3-4-1. 구현 방법 — 시뮬레이션 매핑 *(v4.4 신규)*

물리적 검증 공정이 없는 본 프로젝트에서 Qual 검증을 구현하는 방법. 핵심 원리: **시스템이 소비하는 것은 물리적 wafer가 아니라 FDC Trace 데이터**이므로, 검증 공정의 "출력(데이터)"을 시뮬레이터가 생성하면 파이프라인 코드는 실제와 동일하게 구현된다. 시뮬레이터 상세 명세(3층 구조, 시나리오 YAML, Scorecard): `docs/시뮬레이터_스펙_v1.md`

| 구분 | 항목 |
|------|------|
| **진짜 (실제 구현·실행)** | AE 추론·재학습, Nelson 판정, TTTM 계산, C65 예측, 판정 기준 4종 계산, 승인 워크플로, 실력치 재산정 — 전부 실동작. **Qual 전용 코드 경로 없음** (평소와 같은 Pipeline A/B/C가 처리) |
| **모사 (시뮬레이터가 데이터로 대체)** | PM이라는 물리 이벤트(C33 리셋 + 분포 이동 주입), Qual wafer의 물리적 처리(→ trace 생성), 물리 계측(→ C65 라벨 생성) |

**구현 흐름**:

```
시뮬레이터 (src/simulator — PM/PO 소유):
  ① PM 시나리오 발생 — C33 리셋 + PM 후 센서 분포 이동 주입
     (이동 파라미터는 train 데이터의 실제 PM 구간(C33 리셋 전후)에서 추정 — 실데이터 근거)
  ② chamber 상태 → QUAL 모드, Qual wafer N장을 is_qual=true 플래그와 함께 fdc.raw 발행
     ※ is_qual 필드 추가는 스키마 변경 — 소비자(A, B) 리뷰 필요 (헌법 2-2)

파이프라인 (전부 기존 코드 경로):
  ③ A: C65 예측 / B: AE Score + TTTM / C: Nelson 판정
  ④ 판정 기준 4종 산출 → Dashboard(S7) 제시 → 엔지니어 승인
  ⑤ 승인 시 그 Qual trace로 AE 실제 재학습 + 실력치 실제 재산정
```

**"판정이 틀릴 수 있지 않나?"에 대한 답 — Ground Truth와 4중 안전장치**:

오판(문제 있는데 통과 = 미탐 / 없는데 미통과 = 오탐)은 실재하는 위험이며, 본 설계는 이를 **측정 대상이자 전제**로 다룬다.

1. **Ground Truth는 시뮬레이터가 보유**: 시뮬레이터가 "PM 성공"(분포가 새 정상으로 안착) / "PM 실패"(오염·오조립 잔류 거동) 시나리오를 **정답을 정해 놓고 생성**하므로, Qual 판정 로직의 정판정/미탐/오탐율을 정량 평가할 수 있다 (통과·미통과 시나리오 2종 이상 필수)
2. **판정 기준의 상호보완**: TTTM은 모델이 아니라 **정상 챔버와의 상대 비교**라 "모델이 낡았다" 문제에서 자유롭고, AE는 옛 정상 기준이므로 "극단적 이상 아님"의 느슨한 기준(<0.2)으로만 사용 — 서로 다른 실패 모드를 커버
3. **자동 판정이 아닌 HITL**: 기준 4종은 판정이 아니라 판단 재료. 최종 통과는 엔지니어가 결정 — 판정기의 불확실성을 전제한 게이트 설계
4. **오판의 비대칭 처리**: 미탐(문제 있는데 통과)은 2차 방어(운전 중 drift 알람 → Incident, 직전 Qual 이력 첨부)가 받고, 오탐(없는데 미통과)의 비용은 추가 Qual 몇 장 + 다운타임 연장으로 작음 — 보수적 기준 설정이 정당화됨

> **한계 명시**: 실제 Fab의 Qual은 물리 계측(CD, 파티클)을 포함한다. 본 시스템은 FDC 기반 판정만 구현하며 계측 판정은 C65 Proxy로 대체한다 — 본 시스템의 포지션은 "물리 계측의 대체"가 아니라 **"계측 이전의 1차 게이트"**다.

---

## 4. FDC 실력치 보정 Controller 상세 *(v4.3 전면 개정)*

### 4-0. 실력치(Baseline)란

FDC 시스템은 센서마다 "이 Chamber가 평소에 실제로 내는 값의 정상 범위"(= 실력치, 관리 기준선)를 보유하며, 이를 벗어나면 알람/인터락이 발동한다. PM 등으로 Chamber의 정상 상태가 이동하면 실력치가 낡아 **가성알람이 폭증**한다. 이때 조치는 Recipe 변경(금지)이 아니라 ① **실력치 재설정**(장비 정상, 기준만 갱신), ② **정비 조치**(실제 열화), ③ **이상 wafer 스크랩**이다.

실력치 **관리 단위는 Chamber × Recipe × Step** 이다 *(멘토 확인 완료 — 2026-07-03)*. 단, **Window(과도/정착)는 관리 축이 아니라 지표 추출 계층의 처리**다: summary indicator를 산출할 때 Step 4 점화 과도 구간(첫 9초, C42=1)과 정착 구간(C42=0)을 분리해 추출한다 — 과도 구간은 출렁이는 게 정상이므로 분리하지 않으면 가성알람(점화 때마다 알람) 또는 미탐(과도까지 포용하는 넓은 한계) 중 하나를 강요당한다. 현업 FDC도 summary 값을 step 내 안정 구간에서 추출하는 구조라 이와 정합. (DB의 `sensor_window` 컬럼은 "어느 윈도우에서 추출한 지표인가"의 식별자)

### 4-1. 실력치 재산정 — 2트랙 (멘토 확인 완료, 2026-07-03)

**현업 방식 (멘토 답변)**: "타겟과 스펙을 현실화할 때 **이전 3개월의 평균 ± 3σ 기준으로 자동 리캘리브레이션**된다." — 산식은 EWMA(점진 가중)가 아니라 **rolling window 단순 통계 재계산**이며, 정기 갱신은 자동이다. 이를 첫 피드백("이상 대응 재설정은 엔지니어 확인")과 결합하면 성격이 다른 2트랙이 된다:

```
공통 산식:  center_new  = mean( 최근 N 표본 )        # rolling window — EWMA 아님 (멘토 확인)
            UCL/LCL_new = center_new ± 3σ( 최근 N 표본 )
            ※ Chamber × Recipe × Step 단위 (4-0), 지표는 window(과도/정착)별 추출

트랙 ① 정기 리캘리브레이션 (자동 — 현업 방식):
  - 트리거: 주기적, 무이상 상태에서 배경 실행
  - 승인: 불요 — 단 ⓐ 변동폭 상한(config) 이내일 때만 자동 적용,
    ⓑ limit_corrections에 이력 기록 필수(trigger_type='periodic'), ⓒ 상한 초과 시 승인 요청으로 전환
  - "이전 3개월"의 스케일 환산: 시뮬 환경에서는 최근 N wafer (config)

트랙 ② 이상 대응 재설정 (승인 필수 — 기존 설계 유지):
  - 트리거: Incident (Nelson/TTTM/Drift 알람) → "기준선 노후" 가설 채택 시
  - 표본: 의심 윈도우 제외한 최근 정상 wafer
  - 안전 장치: 1회 재설정 폭 상한 / Seasoning(PM 직후 첫 N wafer) 제외 / 섀도 평가 통과 후 정식 반영
```

트랙 ③으로 **Qual 트리거 재산정**(PM 후 — 3-4절, 승인 필수)까지 합치면 실력치 갱신 경로는 총 3개이며, 모두 동일 산식·동일 이력 테이블(limit_corrections)을 공유한다. 현업 정합 근거: 멘토 답변 + 특허 US10418289("클리닝 후 정상 wafer 로그로 한계 재계산").

### 4-2. Limit Simulation

실력치 재설정안 적용 시 예상 효과를 사전 추정하여, 엔지니어가 승인 판단 시 참고할 수 있도록 함.

```python
def simulate_limit_adjustment(
    current_limits: LimitParams,
    proposed_offset: LimitOffset,
    recent_traces: List[TraceStats],
    historical_outcomes: List[CaseOutcome]
) -> SimulationResult:
    """재설정 적용 시 예상 가성알람 감소율, 미탐(true alarm masking) 위험 추정"""
    ...
```

### 4-3. Interlock & Wafer Disposition — 의심 윈도우 모델 *(v4.3 신규)*

인터락 발생 시 wafer 처분 흐름. Etch 이상 wafer는 rework이 불가하므로 **스크랩이 원칙**이다.

**인터락의 실제 동작**: 진행 중인 run은 안전 문제(Arc, Pressure 급등 등 hard interlock)가 아닌 한 끝까지 진행시킨다 (Etch 도중 중단 = 해당 wafer 100% 스크랩 + Plasma 비정상 종료로 챔버 손상 위험). 인터락의 실체는 **Chamber inhibit — 해당 챔버에 다음 wafer 투입 차단**이다. FDC Trace는 run 완료 후 집계·분석되므로, 본 시스템이 모사하는 동작은 "처리 중단"이 아니라 **"챔버 투입 차단 + 이미 처리된 wafer 사후 격리"**다.

**의심 윈도우(Suspect Window) 정규화**: "wafer 1장 / 구간 / LOT"은 서로 다른 인터락 종류가 아니라 같은 함수의 다른 출력이다. 모든 alert는 `(chamber_id, window_start, window_end)`를 계산하고, HOLD 대상은 윈도우에서 자동 도출된다.

| 신호 유형 | 윈도우 계산 | HOLD 결과 |
|---|---|---|
| N1 단발 (±3σ 1점) | 해당 wafer 1장 | wafer 1장 |
| N2/N3/N4 추세 | **추세 시작 wafer ~ 현재 (소급)** | 구간 (6~14장+) |
| N5 (PM 직후 2σ) | PM 시점 ~ 현재 | PM 이후 전체 구간 |
| Drift/TTTM 지속 | Drift 감지 시작 ~ 현재 | 장기 구간 — 여러 LOT에 걸침 |
| Critical + 원인 불명 | 마지막 정상 확인 시점 ~ 현재 | 보수적 최대 윈도우 |

**계층별 역할**:

- **Wafer HOLD** = 윈도우 안의 wafer들 (계산 결과)
- **LOT HOLD** = 결정이 아니라 **파생** — 윈도우 내 wafer가 속한 LOT은 물류(MES) 단위로 자동 hold. 별도 판단 로직 없음
- **Chamber inhibit** = 윈도우와 무관, **severity로만 결정** (Critical → 즉시 자동, Warning → Supervisor 판단)
- **SCRAP/RELEASE** = HOLD는 윈도우 단위로 잡되, **판정은 wafer별 개별** (wafer당 $15k~22k — VM 예측·SPC 근거로 25장 중 3장만 버릴지 결정하는 정밀도가 곧 비용 절감)

```
인터락/Alert 발생 → 의심 윈도우 계산
    → 윈도우 내 wafer HOLD (+ 소속 LOT 자동 hold)
    → Critical이면 chamber inhibit (신규 투입 차단)
    → wafer별 VM 예측(C65) + SPC 근거로 Disposition 권고 생성
    → 엔지니어 승인
    → wafer별 SCRAP / RELEASE 개별 판정 + inhibit 해제 여부
    → Disposition 이력 저장 (wafer_dispositions, incident_id 연결)
```

### 4-4. Human-in-the-Loop Gate — 멘토 피드백 반영 표준 플로우

```
인터락/Alert 발생 (fdc.alert)
    → AI 분석: TTTM·SPC 관점 이상 예측 및 원인 추정
    → 통합 Agent Service(C): 리포트 3종 병렬 — Recipe 튜닝·실력치 수정·정비 조치 (수치=B 엔진 API, 근거=C RAG) *(4인 재편 2026-07-03 — 구 C·D 통합)*
    → Supervisor Agent 통합 → 승인용 Brief
    → 엔지니어 확인 → 승인/거절/수정
    → 조치 실행 (실력치 반영 / 정비 / 스크랩)
    → 조치 내용 저장 (approval_records)
    → CT 재학습 Trigger → 모델 갱신
```

Supervisor의 핵심 판단은 **4지선다**다 *(v4.5 — Recipe R2R 부활)*: ① **진짜 장비 이상인가** (→ 정비 + 스크랩), ② **공정 조건이 이탈했는가** (→ Recipe R2R — SHAP 기반 튜닝안), ③ **기준선이 낡은 것인가** (→ 실력치 재설정), ④ **모두 아닌가** (→ 공정 검토 에스컬레이션). 경쟁 가설들을 병렬 분석하고 근거(SHAP, TTTM, Drift Score, 과거 사례)로 판정하되, 어느 가설도 성립하지 않으면 ④로 빠진다.

**공정 검토 에스컬레이션 (Process Review Escalation)** *(v4.5 개정)*: Recipe R2R·실력치 재설정·정비 어느 것으로도 해결되지 않는 사건 — 반복 조치에도 재발, 다중 Chamber 동시 열화 등 — 은 시스템의 조치 범위를 벗어나는 문제다. 이 경우 **"판정 불가 + 근거 패키지"**(사건 이력, 시도된 조치와 결과, Evidence Card)를 정리한 에스컬레이션 리포트를 생성하여 공정팀에 전달한다. "세 옵션 모두 아닌 경우"의 라우팅 구멍이 메워진다 (정의된 동작 100% 원칙). *(v4.3의 "Recipe 변경안 제시 금지"는 v4.5에서 폐기 — Recipe R2R은 이제 옵션①)*

**수정 승인(Modify-and-Approve)** *(v4.3)*: 현업 엔지니어는 승인/거절만 하지 않는다 — **제안값을 수정해서 승인**한다 (예: "+4.1%는 과하고 +3.0%로"). 수정 승인 시 원안과 수정값을 모두 `approval_records`에 기록한다. "AI 제안 vs 엔지니어 보정"의 차이 자체가 최고 품질의 학습 데이터가 된다.

### 4-5. 신호→조치 라우팅 매트릭스 *(v4.3 신규)*

조치의 종류는 고정(실력치 재설정 / 정비 / Disposition / inhibit / 모니터링)이며, 룰은 조치를 결정하는 게 아니라 **가설에 가중치를 주는 신호**다. 모든 신호 조합에 정의된 동작이 있어야 한다 (undefined behavior 제로 — 서비스 완전성 원칙).

| 신호 | 패턴 성격 | 초기 가설 기울기 | 자동 동작 (승인 불요) | 승인 요청 |
|---|---|---|---|---|
| N1 (±3σ 급변) | 돌발 (Arc, Leak) | 진성 이상 → 정비+스크랩 우세 | 즉시 inhibit + wafer HOLD | O |
| N2/N3 (연속 추세) | 점진 drift | PM Flag 있으면 실력치, 없으면 정비(마모) | 소급 HOLD | O |
| N4 (증감 반복) | Oscillation (Matching/Hunting) | 정비 우세 | 소급 HOLD | O |
| N5 (PM 직후 2σ) | Seasoning 불충분 / 기준선 노후 | 실력치 우세 | PM 이후 구간 HOLD | O |
| N6 (4/5점 1σ) | Aging 초기 징후 | 모니터링 | 없음 | X (Info 기록만) |
| N7 (15점 1σ 이내) | Chamber Mismatch | TTTM 분석 연계 | 없음 | X (Info 기록만) |
| N8 (8점 1σ 밖) | 두 Population 혼재 | TTTM 분석 연계 → 실력치/정비 판단 | 없음 | O (Warning) |
| TTTM > 2.0 / > 3.0 | 챔버 간 괴리 | 크기·방향·PM 이력에 따라 양쪽 | > 3.0 시 HOLD | O |
| Drift Score 초과 | 분포 이동 | PM 근접도에 따라 양쪽 | 구간 HOLD | O |
| **반복 재발 / 다중 Chamber 동시 이상** *(v4.3)* | 조치 무효 — 공정 조건 의심 | **공정 검토 에스컬레이션** (4-4절) | Incident Reopen + HOLD 유지 | O (에스컬레이션 리포트) |

이 표는 코드가 아니라 **라우팅 설정(config)** 으로 구현한다. 룰별 세부 판단("어떤 룰이면 어떤 조치")은 if-else가 아니라 **Historical Case DB 안에 데이터로 축적**되고, Supervisor가 검색된 근거로 가설을 선택한다 — 룰이 늘어나면 코드가 아니라 사례 데이터가 늘어나는 구조.

---

## 5. RAG 기반 Manufacturing Knowledge Layer *(v4 신규)*

### 5-1. RAG 지식베이스 5타입 분리 *(2026-07-07 개정 — C 제안 채택: Recipe R2R 부활(v4.5)에 따라 recipe_r2r 이력 타입 추가)*

단순 Error Manual RAG를 넘어, **제조 운영 지식베이스**를 5타입으로 분리하여 검색 정확도와 설명력을 높인다. kb_type은 논리 분류 라벨이며, **물리 저장·검색은 2원화**한다: 문서·사례형은 Qdrant 벡터(의미 검색), 이력형은 PostgreSQL(정확 필터·통계 — 운영 테이블 겸용).

| kb_type | 내용 | 활용 목적 | 물리 저장 |
|--------|------|-----------|-----------|
| **error_manual** | 장비 알람, Trouble Shooting Guide, 조치 매뉴얼 | ErrorOccurred 대응 | Qdrant 컬렉션 |
| **process_knowledge** | Etch 공정 원리, 센서 의미, 파라미터 영향 | 설명 생성 및 원인 추론 | Qdrant 컬렉션 |
| **historical_case** | 과거 SPC 위반, Outlier, 조치 결과 사례 | 유사 Case 검색 + 성공률 통계 | **양쪽** — Qdrant(서술 유사검색) + PG `historical_cases`(통계) |
| **limit_change_log** | 과거 실력치 변경 이력, 승인자, 가성알람 감소 실적 | 보정안 신뢰도 판단 | PG `limit_corrections` 겸용 (벡터 없음 — 센서·챔버 정확필터) |
| **recipe_r2r** *(신규)* | 과거 Recipe setpoint 변경 이력·효과 검증 결과 | Recipe 튜닝안 신뢰도 판단 | PG `recipe_corrections` 겸용 (벡터 없음) |

> Qdrant 컬렉션은 **3개**(error_manual / process_knowledge / historical_case)이며 헌법 4-2에 따라 `_v1` 버전 접미사로 생성한다. Qdrant payload에는 페어링 키(`doc_id` 또는 `case_id`)를 필수 포함하여 PG 메타(`rag_documents`)·통계(`historical_cases`)와 조인 가능해야 한다.

**Cold Start 시딩** *(v4.3)*: 현업 투입 초기에는 Historical Case DB가 비어 있어 RAG 근거가 빈약하다. 시뮬레이션으로 생성한 이상 시나리오 사례 + 공정 지식 문서(센서 원리, 파라미터 영향)를 **초기 시딩**하여 첫날부터 Evidence Card가 동작하도록 한다. 이후 approval_records 축적으로 실사례가 시딩 사례를 점진 대체한다.

### 5-2. 이벤트 기반 자동 RAG 검색

RAG는 사용자 질문 시뿐 아니라, **이상 이벤트 발생 시 자동으로 검색**한다. 검색은 Incident 단위로 1회 수행하며(6-0절), Qdrant payload 필터(equipment/chamber/sensor/rule_id/PM 근접도)로 후보를 좁힌 뒤 벡터 유사도로 랭킹한다.

```
SPCViolationDetected / OutlierDetected / LimitCorrectionRecommended
    ↓
Agent receives event
    ↓
이벤트 Payload 기반 RAG Query 자동 생성
    ↓
Historical Case DB 검색 → Error Manual DB 검색 → Limit Change Log 검색
    ↓
Evidence Card 생성 → 승인용 Brief 생성
```

RAG Query는 이벤트 Payload 기반으로 자동 생성한다:

```json
{
  "event_type": "SPCViolationDetected",
  "chamber_id": "E14_3",
  "equipment_id": "EQ_01",
  "rule_id": "N3",
  "sensor": "RF_Vdc",
  "trend": "6 points increasing",
  "tttm_reference": "E14_1",
  "scale_difference_pct": 4.1,
  "predicted_c65_increase": 12.7
}
```

→ Agent가 생성하는 검색 질의 예시:

```
Bay_A E14_3 RF_Vdc Nelson N3 상승 패턴 유사 사례
RF_Vdc scale difference 4% Etch CD 불량 관련 조치
PM 이후 RF_Vdc 실력치 재설정 성공 사례
```

### 5-3. Evidence Card — 구조화된 근거 제시

RAG 결과를 **Evidence Card** 형태로 구조화하여, "왜 이 조치를 믿어야 하는가"에 답한다.

```json
{
  "evidence_card_id": "CASE-2025-09-ETCH-014",
  "similarity_score": 0.91,
  "matched_conditions": {
    "sensor": "RF_Vdc",
    "nelson_rule": "N3",
    "scale_difference_pct": 3.8,
    "equipment_id": "EQ_01"
  },
  "historical_action": {
    "action_type": "limit_correction",
    "param": "RF_Vdc_limit_center",
    "delta_pct": +4.0
  },
  "outcome": {
    "false_alarm_reduction_pct": 62.0,
    "missed_detection": false,
    "success": true
  },
  "source": "Limit Change Log / 2025-09"
}
```

---

## 6. AI Agent Decision Support Layer *(v4 신규)*

### 6-0. Incident 단위 처리 — Alarm Storm 방지 *(v4.3 신규)*

현업에서는 하나의 근본 원인(예: PM 후 상태 이동)이 **여러 센서·여러 룰에서 동시에 알람**을 터뜨린다. 알람을 개별 처리하면 엔지니어에게 승인 요청이 10건씩 쌓이는 시스템이 되고, 그런 시스템은 현장에서 외면당한다.

- **그룹핑 규칙**: 같은 `chamber_id` + 겹치는 시간창의 알람들은 하나의 **Incident**로 묶는다 (`incident_id` 부여)
- **승인 요청은 Incident당 1건**: Agent 분석·RAG 검색·Supervisor Brief·LangGraph thread가 모두 Incident 단위로 생성 (`thread_id` = `incident_id`)
- Incident 진행 중 동일 chamber에서 추가 알람 발생 시: 새 승인 요청을 만들지 않고 **기존 Incident에 병합**, 의심 윈도우 확장 및 Brief 갱신
- 개별 알람은 전부 `spc_violations`에 기록 유지 (감사 추적) — 그룹핑은 처리 단위의 문제이지 기록 생략이 아님

**Incident Lifecycle** *(v4.3)*: 승인이 사건의 끝이 아니다. 조치 후 효과 확인까지가 사건의 수명이다.

```
open → analyzing → pending(승인 대기) → actioned(조치 실행)
     → verifying(효과 확인: 실력치는 섀도 평가, 정비는 재가동 후 N wafer 관찰)
     → closed(효과 확인됨)
     └→ reopened(verifying 중 동일 패턴 재발)
          → 재발 카운트 증가, 기존 이력 승계
          → 반복 재발 시 공정 검토 에스컬레이션 (4-4절)
```

종료 기준(closure criteria)이 없으면 시스템은 "처리됐다"를 판단할 수 없다. verifying 통과 조건은 조치 유형별로 정의한다 (실력치: 섀도 평가 N wafer 미탐 0건 + 가성알람 감소 확인 / 정비: 재가동 후 N wafer 정상 + TTTM 정상 범위 복귀).

**승인 큐 운영** *(v4.3)*: Pending 목록은 ① **우선순위 정렬** — severity × 예상 비용(HOLD 물량) 기준, Critical이 Warning에 묻히지 않게, ② **담당자 지정(claim)** — 사건을 잡은 엔지니어를 기록해 두 명이 동시에 같은 사건을 파는 것 방지, ③ 대시보드 알림(badge) 기본, Slack/Teams 연동은 Could.

### 6-1. Agent 역할 구성

AI Agent는 **Tool-Using Agent**로 설계한다. 단순 LLM 답변이 아니라, 제조 데이터 파이프라인 위에서 행동하는 AI이다.

| Agent (논리적 역할) | 경로 | 역할 | 입력 | 출력 |
|---------------------|------|------|------|------|
| **Severity Assessor** | **fast-path (동기, LLM 미사용)** | 하드알람/재설정 폭 상한/SHAP·TTTM 기반 **심각도 평가** + 자동 동작(HOLD/inhibit) + Incident 그룹핑 + **리포트 3종 병렬 탐색** 개시 | `fdc.alert` + SHAP Top-3 + TTTM Score | 심각도 점수 + L5·L6 병렬 트리거 (sub-second) |
| **Root Cause Analysis** | slow-path (비동기) | C의 판정 사실 + Process Knowledge DB를 결합하여 **원인 후보 추론** | `spc_violations` + `tttm_comparisons` + Process Knowledge DB (RAG) | 원인 후보 목록 + 근거 |
| **Recipe Tuning Agent (통합 Agent Service tool① — C)** *(v4.5 신설)* | slow-path (비동기) | **옵션① Recipe 튜닝안** — SHAP 축→손잡이(C1/C4/C5) 변환 + delta(수치=B5-4 엔진) + Process KB 근거 | Process Knowledge DB + Recipe R2R Log (RAG) | Recipe 튜닝 권고안 + Evidence Card |
| **Limit Correction Agent (통합 Agent Service tool② — C)** | slow-path (비동기) | **옵션② 실력치 재설정안** 탐색 + 과거 성공률 검색 (수치=B 엔진) | Historical Case DB + Limit Change Log DB (RAG) | 실력치 재설정 권고안 + Evidence Card |
| **Maintenance Agent (통합 Agent Service tool③ — C)** *(4인 재편 — 구 팀원 D)* | slow-path (비동기) | **옵션③ 설비 점검안** 탐색 | Error Manual DB (RAG) | 정비 매뉴얼 권고안 + Evidence Card |
| **Supervisor Agent** | slow-path (비동기) | 세 옵션의 **순위를 매기고** 하나의 Brief로 통합 — 4지선다 판정("진짜 이상 / 공정 조건 이탈(Recipe) / 기준선 노후 / 모두 아님→에스컬레이션") | Root Cause + 옵션 3종 출력 | Approval Brief (순위 + 각 옵션 근거 + Disposition 권고) → 엔지니어가 최종 선택 |
| **CT Decision Advisor** | slow-path (비동기) | 재학습 필요성 판단 | Drift, Valid 성능 | CT Trigger 제안 |

> **핵심** *(v4.5 — 4인 재편·Recipe R2R 반영)*: 통합 Agent Service(C)의 **tool 3종(Recipe·실력치·정비)은 항상 병렬 실행**된다. Supervisor가 순위를 매기지만 세 옵션 모두 Evidence Card와 함께 엔지니어에게 제시되어 **엔지니어가 최종 선택**한다.

> **구현 참고**: 실제 구현은 하나의 Agent Service 안에 `tools`로 구현하고, 발표에서는 논리적 Agent 역할로 설명한다. Severity Assessor는 LLM 미사용(규칙 기반), 나머지는 RAG+LLM 기반.

### 6-2. Agent가 사용할 Tool 목록

| Tool | 기능 | 예시 호출 |
|------|------|-----------|
| `get_recent_trace()` | 최근 FDC Trace 조회 | 특정 Chamber의 최근 30 wafer |
| `get_spc_status()` | Nelson Rule 위반 내역 조회 | N1~N8 평가 결과 |
| `get_tttm_comparison()` | fleet 분포 대비 센서 Scale 비교 | SIM_CH_3 vs fleet median |
| `get_shap_top_features()` | 예측 모델의 주요 기여 인자 조회 | C65 예측 Top-3 |
| `search_similar_cases()` | 과거 유사 사례 검색 (RAG) | RF_Vdc N3 pattern |
| `search_manual()` | Trouble Shooting 문서 검색 (RAG) | RF bias 관련 매뉴얼 |
| `simulate_limit_adjustment()` | 실력치 재설정 시 가성알람 변화·미탐 위험 추정 | RF_Vdc 한계 +4% 재설정 |
| `recommend_disposition()` | VM 예측 기반 wafer 스크랩/해제 권고 | HOLD 중 LOT 판정 |
| `create_approval_ticket()` | 승인 요청 티켓 생성 | engineer review 필요 |

### 6-3. Agent Report 출력 예시

```json
  "agent_report": {
    "incident_id": "INC-20260702-E14_3-001",
    "severity_score": 85,
    "suspected_root_causes": ["post-PM chamber condition shift", "focus ring wear (early)"],
    "parallel_options": {
      "limit_option": {
        "action": "RF_Vdc 실력치 +4.1% 재설정 (PM 후 정상 이동 판정)",
        "evidence_cards": [{"case_id": "CASE-2025-09", "result": "가성알람 62% 감소, 미탐 없음"}],
        "feasibility": "HIGH (다운타임 없음)"
      },
      "manual_option": {
        "action": "Inspect and replace ESC O-ring",
        "evidence_cards": [{"manual_id": "MAN-ETCH-004", "result": "Resolves severe impedance drift"}],
        "feasibility": "LOW (requires downtime)"
      }
    },
    "wafer_disposition": {
      "held_lot": "LOT_0681",
      "recommendation": "RELEASE",
      "reason": "VM 예측 C65 정상 범위 — 가성알람 판정"
    },
    "supervisor_recommendation": {
      "selected_option": "limit_option",
      "reason": "PM Flag(C33) 직후 + Drift 패턴이 과거 PM 후 정상 이동 사례와 일치. 진성 열화 증거 부족.",
      "confidence": 0.88
    }
  }
}
```

### 6-4. AI Agent 능동 개입 시나리오 — 핵심 시나리오

#### 시나리오: RF_Vdc 이상 상승에 대한 Agentic 실력치 보정 승인 지원

1. **SensorDataReceived** — E14_3 Chamber에서 FDC Trace 유입

2. **WaferPredicted** — LightGBM 모델이 C65 증가 가능성 예측, SHAP Top-3: RF_Vdc, Pressure, ESC_Temp

3. **SPCViolationDetected** — RF_Vdc에서 Nelson Rule N3 감지, 6개 wafer 연속 상승 → 해당 wafer **HOLD**

4. **TTTM Analysis** — Reference Chamber E14_1 대비 RF_Vdc 평균 Scale +4.1%. PM Flag(C33) 12 wafer 전 발생 확인

5. **LimitCorrectionRecommended** — 실력치 엔진이 rolling mean±3σ로 RF_Vdc 실력치 +4.1% 재설정안 산출 (PM 후 정상 이동 가설)

6. **AgentAnalysisRequested** — AI Agent가 이벤트 Context 수집, 팀원 C(실력치)·D(정비) 병렬 분석

7. **EvidenceRetrieved** — 과거 유사 Case 3건 검색. 가장 유사한 Case: PM 후 동일 패턴에서 실력치 재설정으로 가성알람 62% 감소, 미탐 없음. 반면 정비(ESC O-ring) Case는 impedance drift가 심각 수준일 때만 유효

8. **AgentReportGenerated** — Supervisor가 "기준선 노후" 가설 채택, HOLD wafer는 VM 예측 정상으로 RELEASE 권고

9. **ApprovalRequested** — 엔지니어에게 승인 요청 (LangGraph interrupt → Pending)

10. **CorrectionApplied 또는 Rejected** — 승인 시 섀도 평가(N wafer) 후 실력치 정식 반영 + CT 재학습 Trigger, 반려 시 사유 저장 후 RAG 학습 데이터로 축적

> **Agent의 대시보드 메시지 예시**:
>
> "엔지니어님, E14_3 Chamber의 RF_Vdc에서 Nelson Rule N3(6점 연속 상승 추세)이 감지되었습니다. Reference Chamber E14_1 대비 RF_Vdc가 **4.1% 높으며**, TTTM Score가 경고 수준(2.3)에 도달했습니다. 다만 **12 wafer 전 PM이 수행**되었고, 상승 패턴이 과거 PM 후 정상 이동 사례와 일치합니다.
>
> **제안**: RF_Vdc 실력치를 +4.1% 재설정할 것을 권고합니다. 2025-09월 E14_1 Chamber의 동일 패턴에서 실력치 재설정으로 가성알람이 62% 감소했고 미탐은 없었습니다(성공률 92%, 26건 중 24건). HOLD 중인 wafer는 VM 예측이 정상 범위로 **RELEASE**를 권고합니다.
>
> **대안**: 진성 열화 가능성 배제를 위해 ESC O-ring 점검(다운타임 필요)도 병기합니다.
>
> **SHAP 분석**: C65 예측의 주요 기여 인자는 RF Vdc(58%), Chamber Pressure(27%), ESC_Temp(15%)입니다."

### 6-5. 승인 상태머신 — LangGraph 구현 *(v4.3 신규, 팀원 제안 채택)*

Pending/Approve/Reject 상태머신을 별도 백엔드로 개발하지 않고, **LangGraph의 그래프 정의 + 체크포인터**로 흡수한다.

**핵심 메커니즘**: LangGraph는 노드 실행마다 그래프 상태를 체크포인터에 자동 저장한다. 노드 안에서 `interrupt()`를 호출하면 실행이 멈추고 상태가 저장된 채 리턴된다 — 이 "멈춰서 저장된 상태"가 곧 **Pending**이다. Approve/Reject는 같은 `thread_id`로 `Command(resume={"action": ...})`를 재invoke하는 것이며, 조건부 엣지가 다음 노드로 자동 라우팅한다.

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt, Command

def merge_reports(state):
    """Supervisor: C(실력치)·D(정비) 리포트 통합 + Disposition 권고"""
    ...
    return {"report": report}

def approval_gate(state):
    """단일 승인 게이트 (헌법 1-4: 게이트는 Supervisor 뒤 한 곳만)"""
    decision = interrupt({"type": "approval", "report": state["report"]})
    return {"decision": decision["action"]}

def apply_correction(state):
    """승인 시: 실력치 반영 / 정비 조치 기록 / wafer Disposition 확정
    → approval_records 기록 → CT 재학습 Trigger 발행"""
    ...

def reject_and_learn(state):
    """반려 시: 반려 사유 approval_records 기록 → RAG 인덱스 재투입"""
    ...

graph = StateGraph(State)
graph.add_node("merge_reports", merge_reports)
graph.add_node("approval_gate", approval_gate)
graph.add_node("apply_correction", apply_correction)
graph.add_node("reject_and_learn", reject_and_learn)
graph.add_edge("merge_reports", "approval_gate")
graph.add_conditional_edges(
    "approval_gate", lambda s: s["decision"],
    {"approve": "apply_correction", "reject": "reject_and_learn"}
)
app = graph.compile(checkpointer=PostgresSaver.from_conn_string(DB_URI))
```

```python
# 실행 흐름
config = {"configurable": {"thread_id": "incident-12345"}}  # thread_id = incident_id
app.invoke(initial_state, config)                           # approval_gate에서 멈춤 (= Pending)
# 승인/반려/수정 API 호출 시 같은 thread_id로 재개
app.invoke(Command(resume={"action": "approve"}), config)   # → apply_correction
app.invoke(Command(resume={"action": "approve",
                           "modified_value": +3.0}), config) # 수정 승인 — 원안+수정값 모두 기록
app.invoke(Command(resume={"action": "reject",
                           "reason": "..."}), config)        # → reject_and_learn
```

**설계 원칙 및 제약**:

| 항목 | 결정 |
|------|------|
| 체크포인터 | 처음부터 **Postgres 체크포인터** 사용 (docker-compose에 Postgres 이미 존재, SQLite 마이그레이션 불필요) |
| `approval_records` | 체크포인터가 대체하지 **않는다**. interrupt 시 Pending row 생성, resume 시 결과·사유 업데이트 — 감사 추적 + RAG 학습 자산 + 대시보드 Pending 목록 조회를 모두 이 테이블로 해결 |
| 게이트 위치 | Supervisor 통합 후 **단일 게이트** (개별 Agent의 직접 승인 요청 금지 — 헌법 1-4) |
| Kafka 이벤트 | interrupt 시 `ApprovalRequested`, resume 시 `ApprovalCompleted` 발행 유지 (Event Contract 준수) |
| Pending 타임아웃 | TTL 정책 필요 (예: 24h 무응답 시 재알림/에스컬레이션) |
| 중복 resume 방어 | resume 전 approval_records 상태 확인 (두 명이 동시 승인 클릭 시 멱등성) |
| 그래프 버전 변경 | 노드 구조 변경 배포 시 기존 Pending 스레드 resume 불가 가능 — 배포 전 Pending 정리 규칙 필요 |

**결론**: 상태 전이 로직 + 커스텀 백엔드가 LangGraph 그래프 정의와 체크포인터로 흡수되고, DB는 기록·조회 전용으로 단순해진다. 반려 시 `reject_and_learn` 노드가 RAG 재투입까지 이어져 기존의 "반려 피드백 재투입 파이프라인"과 그대로 연결된다.

---

## 7. Kafka Event Contract v4

### Kafka Topic 구성 (6개)

```
fdc.raw           # 원천 센서 데이터
fdc.prediction    # VM 예측 결과 + SHAP
fdc.alert         # SPC 위반, Outlier, Drift 이벤트 (인터락 트리거)
fdc.correction    # 실력치 보정 권고·적용 결과, wafer Disposition (구 fdc.r2r)
                  # ★ 팀원 B(SPC)가 구독 — 승인된 실력치(limit_version, effective_from)로 한계 즉시 갱신
fdc.agent         # Agent 분석 요청/결과, Evidence, Approval
fdc.mlops         # CT 재학습 권고 및 실행 결과
```

### 주요 Event Type

| Topic | Event Type | 목적 |
|-------|------------|------|
| `fdc.raw` | `SensorDataReceived` | FDC Trace 유입 |
| `fdc.prediction` | `WaferPredicted` | C65 예측 완료 |
| `fdc.alert` | `SPCViolationDetected` | Nelson Rule 위반 감지 |
| `fdc.alert` | `OutlierDetected` | 이상치 감지 |
| `fdc.alert` | `DriftScoreUpdated` | Drift Score 갱신 |
| `fdc.correction` | `LimitCorrectionRecommended` | 실력치 재설정안 산출 *(v4.3)* |
| `fdc.correction` | `CorrectionApplied` | 실력치 반영 완료 *(v4.3)* |
| `fdc.correction` | `WaferDispositioned` | wafer 스크랩/해제 판정 완료 *(v4.3)* |
| `fdc.agent` | `AgentAnalysisRequested` | Agent 분석 요청 *(v4)* |
| `fdc.agent` | `EvidenceRetrieved` | RAG 검색 결과 저장 *(v4)* |
| `fdc.agent` | `AgentReportGenerated` | Agent 분석 리포트 완료 *(v4)* |
| `fdc.agent` | `ApprovalRequested` | 엔지니어 승인 요청 *(v4)* |
| `fdc.agent` | `ApprovalCompleted` | 승인/반려 결과 *(v4)* |
| `fdc.mlops` | `CTDecisionRecommended` | 재학습 권고 *(v4)* |

### JSON Contract 예시

```json
{
  "wafer_info": {
    "wafer_id": "C64_0681",
    "chamber_id": "E14_3",
    "equipment_id": "EQ_01",
    "equipment_type": "ETCH",
    "recipe_id": "ETCH_RCP_042"
  },
  "prediction": {
    "c65_score": 1450.5,
    "risk_level": "CRITICAL",
    "top_3_causes": [
      {"sensor": "RF_Vdc", "contribution": 0.58},
      {"sensor": "Chamber_Pressure", "contribution": 0.27},
      {"sensor": "ESC_Temp", "contribution": 0.15}
    ],
    "spc_flags": [
      {"sensor": "RF_Vdc", "rule": "N3"}
    ]
  },
  "chamber_status": {
    "is_pm_drift_detected": true,
    "anomaly_score": 0.88,
    "days_since_last_pm": 94,
    "tttm_score": 4.1
  },
  "spc_metrics": {
    "tttm_score": 2.3,
    "tttm_reference": "E14_1",
    "active_nelson_rules": [
      {"rule": "N3", "sensor": "RF_Vdc", "severity": "WARNING", "started_at": "2026-06-24T14:30:00Z"}
    ]
  },
  "correction_recommendation": {
    "incident_id": "INC-20260702-E14_3-001",
    "limit_adjustments": [
      {"sensor": "RF_Vdc", "limit_center_current": 200.0, "limit_center_proposed": 208.2,
       "ucl_proposed": 214.4, "lcl_proposed": 202.0, "unit": "V", "delta_pct": 4.1,
       "limit_version": "v2", "effective_from": "2026-07-02T16:00:00Z"}
    ],
    "predicted_tttm_after": 1.2,
    "predicted_false_alarm_reduction_pct": 62.0,
    "historical_success_rate": 0.92,
    "requires_approval": true,
    "recommended_validation": "Shadow evaluation (30 wafers)"
  },
  "wafer_disposition": {
    "held_wafers": ["C64_0681"],
    "recommendation": "RELEASE",
    "reason": "VM predicted C65 within normal range"
  },
  "agent_report": {
    "incident_id": "INC-20260702-E14_3-001",
    "severity_score": 85,
    "suspected_root_causes": ["post-PM chamber condition shift", "focus ring wear (early)"],
    "parallel_options": {
      "limit_option": {
        "action": "RF_Vdc limit center +4.1% reset",
        "evidence_cards": [{"case_id": "CASE-2025-09-ETCH-014", "result": "False alarms -62%, no missed detection"}]
      },
      "manual_option": {
        "action": "Inspect ESC O-ring",
        "evidence_cards": [{"manual_id": "MAN-ETCH-004", "result": "Resolves drift"}]
      }
    },
    "supervisor_recommendation": {
      "selected_option": "limit_option",
      "reason": "Post-PM shift pattern matches successful past limit corrections; no evidence of true degradation.",
      "confidence": 0.88
    }
  }
}
```

---

## 8. 데이터베이스 설계

### 주요 테이블

| Table | Pipeline | 역할 |
|-------|----------|------|
| `wafer_predictions` | A | Wafer별 C65 예측 결과, SHAP Top-3 |
| `spc_violations` | C | Nelson Rule 위반 이벤트 |
| `tttm_comparisons` | C | Chamber 간 센서 비교 결과 |
| `incidents` | E | 알람 그룹핑 단위 — lifecycle 상태(open~closed/reopened), 재발 카운트, LangGraph `thread_id` 매핑, 담당자(claim) *(v4.3 신규)* |
| `sim_events` | 시뮬레이터 | 주입 이벤트 + Ground Truth 라벨 — Scorecard 채점 전용. **파이프라인은 읽기 금지** (답안지 커닝 방지) *(v4.4 신규, 상세: `docs/시뮬레이터_스펙_v1.md`)* |
| `limit_corrections` | D | 실력치 재설정 제안·적용 이력 (`limit_version`, `effective_from` 포함) *(v4.3, 구 r2r_recommendations)* |
| `wafer_dispositions` | D | wafer HOLD/SCRAP/RELEASE 판정 이력 *(v4.3 신규)* |
| `approval_records` | E | 엔지니어 승인/반려 이력 — 실력치 변경 감사 추적 포함, LangGraph Pending 목록 조회 겸용 *(v4 신규)* |
| `historical_cases` | E | 과거 이상 사례 구조화 데이터 *(v4 신규)* |
| `rag_documents` | E | 매뉴얼/Case 문서 메타데이터 *(v4 신규)* |
| `agent_reports` | E | Agent가 생성한 분석 리포트 *(v4 신규)* |
| `ct_decisions` | MLOps | 재학습 권고 및 실행 결과 |

> **핵심**: `approval_records`가 쌓이면 Agent가 "어떤 추천이 승인/반려되었는가, 반려 사유는 무엇인가, 승인 후 실제 개선되었는가"를 학습할 수 있다. Approval Log 자체가 제조 지식 자산이 된다.

---

# III. Scope · 로드맵 · 정리

## 0. 서비스 설계 원칙 — "작은 인프라, 완전한 서비스" *(v4.3 신규)*

본 프로젝트의 목표는 데모용 MVP가 아니라 **현업에 투입 가능한 수준의 서비스**다. "작게"는 규모의 문제고 "제대로"는 완전성의 문제이며, 이 둘을 구분한다.

**작게 유지하는 것 (규모)**: 단일 Kafka 브로커, docker-compose, React 대시보드(단일 SPA), Chamber 6대, LLM 1종 — 이후 인프라만 교체하면 되는 부분.

**타협하지 않는 것 (완전성)**:

1. **정의된 동작 100%** — 모든 신호 조합(N1~N8 × TTTM × Drift)에 라우팅이 정의됨 (4-5절). undefined behavior 제로
2. **상태머신 완전성** — wafer/chamber/승인의 모든 전이와 엣지 케이스(중복 승인, 대기 중 신규 알람, 반영 실패 롤백) 정의
3. **Incident 단위 처리** — Alarm Storm에서도 승인 요청은 사건당 1건 (6-0절)
4. **수정 승인** — 엔지니어가 제안값을 고쳐 승인하는 현업 기본 UX (4-4절)
5. **안전한 실패** — 판정 불확실 시 보수적 기본값(정비 점검 우선, HOLD 유지), 실력치 재설정 롤백 가능
6. **완전한 기록** — 모든 알람·판정·승인·수정이 감사 추적 가능

## 1. 구현 범위 정의

**Must + Should = 즉시 구현**, **Could = 확장으로 열어둠**.

### 즉시 구현 (Must + Should)

#### Pipeline A — Wafer 단위 C65 예측

| 항목 | 근거 |
|------|------|
| LightGBM + SHAP Top-3 기여 인자 산출 (모델 원본 순위 유지) | 예측의 핵심. SHAP 설명 가능성 우선 |
| Optuna 하이퍼파라미터 최적화 | 예측 정확도 확보 |
| SHAP Top-3 + SPC Flag **병렬 제시** (독립 채널) | 모델 근거와 통계 근거의 비교 가능성 — SHAP 순위 개입 금지 *(v4.4)* |

#### Pipeline B — Chamber 단위 Drift/Outlier 감지

| 항목 | 근거 |
|------|------|
| Autoencoder 기반 Anomaly Detection + Drift Score | PM Drift 감지 핵심 |
| PM 카운터(C33) 리셋 감지 기반 트리거 | PM 이벤트 식별 (C33 = **RF time**, RF 누적 시간 — 리셋 = PM. 멘토 확인 v4.5) |
| TTTM 비교: fleet 분포 대비 센서 Scale 차이 계산 | Chamber Matching의 기초 |

#### Pipeline C — Smart SPC Engine

| 항목 | 근거 |
|------|------|
| Nelson Rules N1~N8 전체 구현 | SPC Engine의 핵심 |
| TTTM 비교 분석 (Chamber 간 센서 Scale 차이) | Chamber Matching의 기초 |
| SPC Dashboard (Control Charts + Nelson View) | 시각화 필수 |

#### Pipeline D — FDC 실력치 보정 Controller *(v4.3 개정)*

| 항목 | 근거 |
|------|------|
| 실력치 재산정 엔진 (rolling mean±3σ, 2트랙) — **모니터링 전 센서 대상** (우선 검증: RF_Vdc) | Closed Loop 증명 + 현업 완전성 (멘토 확인 산식) |
| 신호→조치 라우팅 매트릭스 전체 커버 (N1~N8 × TTTM × Drift) | undefined behavior 제로 |
| Interlock → 의심 윈도우 계산 → wafer HOLD(LOT 파생) → wafer별 SCRAP/RELEASE | 현업 표준 흐름 반영 |
| Incident 그룹핑 (승인 요청 사건당 1건) | Alarm Storm 방지 — 현업 수용성 핵심 |
| Human-in-the-Loop Approval Gate (LangGraph interrupt) + **수정 승인** | 엔지니어 신뢰 확보 전제 |
| Limit Simulation (가성알람 감소율·미탐 위험 추정) + 재설정 롤백 | 재설정안 효과 사전 추정 + 안전한 실패 |
| CT Trigger 이벤트 생성 | 재학습 자동화 연결 |

#### Pipeline E — RAG 기반 AI Agent *(v4 신규)*

| 항목 | 근거 |
|------|------|
| Qdrant 기반 Vector DB (payload 필터 + 벡터 검색) | 유사 Case/Manual 검색 인프라 |
| RAG 기반 유사 Case 검색 | 이벤트 기반 자동 검색 |
| Historical Case DB 초기 시딩 (Cold Start 대응) | 투입 초기 RAG 근거 확보 — 시뮬레이션 생성 사례 + 공정 지식 문서 |
| Error Manual RAG | Trouble Shooting 자동 참조 |
| Agent 기반 승인용 Brief 자동 생성 | Approval Gate의 UX 핵심 |
| Approval Workflow — LangGraph interrupt + Postgres 체크포인터 (승인/수정/반려 + 사유 기록) | 피드백 축적, 상태머신 코드 최소화 |
| Incident Lifecycle (open~verifying~closed/reopened) + 승인 큐 운영(우선순위·claim) | 사건 종료 기준 + 현장 운영성 *(v4.3)* |
| Agent 추천 KPI 대시보드 (채택률/수정률/반려율) | 서비스 신뢰도 실측 + Agent 약점 영역 식별 *(v4.3)* |
| 엔지니어 자연어 질의 (사건 이력·지식 직접 검색) | Tool API 재사용 — 이벤트 기반 검색의 능동형 보완 *(v4.3)* |
| Agent Tool API (get_spc_status, search_similar_cases 등) | Tool-Using Agent 동작 |

#### EAL — Equipment Abstraction Layer

| 항목 | 근거 |
|------|------|
| Schema Registry 구조 설계 + Bay A 3대 Chamber 등록 | EAL 존재 증명 |
| `EquipmentAdapter` 인터페이스 정의 및 Bay A Adapter 구현 | 추상화 동작 증명 |

#### MLOps — CT Loop 2종 *(v4.4: 성격이 정반대인 두 재학습을 분리)*

| CT | 대상 모델 | 개입 방식 | 흐름 | 근거 |
|----|----------|----------|------|------|
| **① LightGBM** | Pipeline A (C65 예측) | **자동 재학습** — 단, 배포는 champion–challenger 통과 시 (가드레일 3) | 실측 Y 도착 → RMSE 채점 → 재학습 (트리거 시점 미정 ※) → 평가 통과 시 MLflow 버전 갱신·배포 | "능동형" 핵심 — Y 예측 재학습에 사람 개입 없음 |
| **② Autoencoder** | Pipeline B (Chamber 이상 감지) | **수동 (엔지니어 승인)** | PM 완료 → Qual Wafer 검증 (3-4절) → Dashboard 제시 → 엔지니어 승인 → AE 재학습 + 실력치 재산정 → MLflow 버전 갱신 | Human-in-the-Loop — PM 후 설비 오프라인 상태에서 정상 여부 판단 필요 |

> **※ CT① 재학습 트리거 (확정 — v4.5, 멘토 제안 시퀀스)**: **PM Cycle(C33=RF time 리셋 간격) 종료 시** sliding window(최근 3개월 상당 = 과거 여러 Cycle) 재학습 → 다음 3 Cycle 예측. Cycle 내에는 Model R2R(Residual Bias 자동 update, 상한 clamp + ct_decisions 기록)로 대응하고, 상한 초과 격차는 재학습이 아니라 **Incident 신호**로 라우팅 (PM 초반 며칠치 데이터만으로는 재학습이 성립하지 않음).

#### Data Engineering

| 항목 | 근거 |
|------|------|
| Airflow 기반 전처리 → Mart 적재 | 데이터 파이프라인 기반 |
| Kafka Producer로 정적 CSV를 초 단위 스트리밍 모사 | 실시간 모사 |

### 확장으로 열어둠 (Could)

향후 시간이 남거나 프로젝트 확장 시 추가할 수 있는 기능.

| 항목 | 이유 |
|------|------|
| 통계 기반 Outlier 탐지 (Pipeline B 보조) | Autoencoder로 확정 (v4.4) — 통계 기반은 보조 수단으로 확장 시 |
| 1D-CNN 모델 | LightGBM + SHAP만으로 충분. Trace 형태가 살아있을 때만 의미 |
| LightGBM + CNN Ensemble | 발표 시 복잡도 증가, 검증 부담 |
| Hotelling T² / Q-statistics (MSPC) | 논문 수준 완성도. 구현 복잡도 높음 |
| ~~Bay-Adaptive EWMA~~ | 삭제 — 멘토 확인으로 EWMA 자체가 폐기됨 (rolling mean±3σ) |
| Bay B(구형) 시뮬레이션 데이터 생성 + Bay-Adaptive Control Limits | KL Divergence 검증 Gate 필요 |
| Multi-Agent 물리적 분리 | 논리적 분리로 충분. 물리 분리는 인프라 부담 |
| Slack/Teams 알림 연동 | UX 확장. 핵심 기능 아님 |
| 승인 권한 모델(RBAC) — 조치 유형별 승인권자 분리 (실력치 vs 스크랩) | 현업 필수이나 팀 데모 환경에서는 역할 1개로 충분. 스키마에 승인자 role 컬럼만 선반영 *(v4.3)* |
| ~~LLM 온프레미스/로컬 전환~~ → **즉시 구현으로 승격** *(2026-07-06 PM 확정)*: 로컬(오픈소스) LLM을 **AWS GPU 인스턴스에 셀프호스팅** (vLLM, 단일 EC2 + Docker Compose — config 합의안 D9~D22) | 외부 LLM API 미사용. 프롬프트 데이터 마스킹만 확장으로 유지 |
| Cross-Bay Chamber Assignment 추천 (Scenario B) | Fleet View 부가 기능 |
| Kafka → Spark 실시간 파이프라인 | Airflow Micro-Batch로 대체 가능 |
| **Inline 계측(검사공정) 연동 — 레벨-모양 분리 완성** *(2026-07-08 추가)* | 현 데이터엔 fail bit(WT, ~2개월 지연)뿐이라 요란 PM 후 라벨 공백 동안 Y 레벨 추적이 늦음. 현업처럼 검사공정(CD-SEM·파티클 인스펙션, 수시간~수일) 신호가 들어오면 레벨은 inline+bias 계층이 빠르게 전담, 모델은 레짐 내 변동(모양)만 학습하는 구조로 완성됨. 데이터 부재로 확장 보류 — 공백 완화는 당장은 C17 전이 프라이어(검증 중)·가한계·재학습 3단이 담당 |

### Won't-have (명시적 배제)

| 항목 | 이유 |
|------|------|
| ~~Recipe 보정을 통한 이상 대응~~ | **v4.5에서 Won't-have 해제** — 멘토 정정("레시피 수정도 서비스에 넣어야 한다"). 무승인 자동 반영 금지는 아래 행으로 유지 |
| Recipe **무승인 자동** 반영 | 완전 자동 Recipe 변경은 매우 위험 — 반드시 승인 게이트 경유 (헌법 1-1) |
| 완전 자동 Closed-Loop 제어 | "Human-in-the-Loop 기반 Semi-Closed Loop"가 현실적 |
| 실력치 **이상 대응** 자동 반영 (승인 생략) | 진짜 이상을 기준선 이동으로 가릴 위험 — 승인 게이트 필수 (정기 리캘리브레이션만 상한 내 자동 — 헌법 1-1 예외) |
| CQRS / Event Sourcing | 아키텍처 복잡도 과도 |
| Safe RL supervisory layer | 연구 수준. 프로젝트 범위 초과 |
| LLM이 직접 제어 파라미터를 임의 생성하는 구조 | 안전성 미확보 |
| Full Fleet Dashboard (10 Chamber+) | 6대면 충분 |

---

## 2. 8주 실행 계획

RAG/Agent를 포함한 5대 Pipeline 구현을 위한 Phase별 Deliverable.
> **진행 현황:** PM/PO가 Phase 1(기획·인프라·ML 모델링 조기 완료)을 **1주차에 조기 완료**하였으며, 2주차(인프라 연동 및 Consumer 뼈대)도 **완료**되었습니다. 이에 따라 팀원 A의 역할이 '실시간 추론 파이프라인(MLOps)'으로 전환되어 3주차부터 코어 로직에 집중합니다.

| Phase | 기간 | 핵심 Deliverable | 진행 상태 |
|-------|------|-----------------|-----------|
| **Phase 1: Data & Baseline** | 1~2주 | FDC CSV 스트리밍, Kafka Topic(6개), DB Schema, 앙상블 모델링(.pkl) 조기 완료, 팀원 A MLOps 추론 뼈대 | ✅ **완료** |
| **Phase 2: SPC/TTTM Core** | 3~4주 | Autoencoder Drift Detection, Nelson Rules N1~N8, TTTM 비교 API, Context Score 산출. *(PM/PO)* 시뮬레이터 확장 1차 — 멀티챔버 복제 + 시나리오 주입기 (`docs/시뮬레이터_스펙_v1.md`) | 🚀 **진행 예정 (3주차 차례)** |
| **Phase 3: RAG & Agent Layer** | 5~6주 | Qdrant Vector DB + 초기 시딩, 유사 Case/Manual RAG 검색(payload 필터), 병렬 Agent(C, D) Tool API, Incident 그룹핑, Supervisor Agent 뼈대. *(PM/PO)* 시뮬레이터 확장 2차 — QUAL 모드 + 시나리오 라이브러리 + Scorecard | 📅 대기 |
| **Phase 4: 실력치 보정 & E2E Demo** | 7~8주 | Supervisor Agent 로직 완성, LangGraph 승인 상태머신(interrupt + Postgres 체크포인터), 실력치 보정·Disposition E2E 통합, 최종 시연 | 📅 대기 |

### Phase별 핵심 증명 사항

```
Phase 1-2 (4주): "Etch FDC로 불량을 예측하고, SPC가 이상을 감지할 수 있다"
  ├─ Pipeline A: C65 Baseline (RMSE, SHAP Top-3)
  ├─ Pipeline B: Autoencoder Drift Detection + TTTM
  ├─ Pipeline C: Nelson Rules N1~N8 + TTTM Dashboard
  └─ EAL: Schema Registry + Bay A Adapter

Phase 3-4 (4주): "FDC → SPC → 실력치 보정 → Agent가 하나의 Closed Loop로 연결된다"
  ├─ Pipeline D: 실력치 재산정(mean±3σ) + Disposition + Approval Gate
  ├─ Pipeline E: RAG 검색 + Agent Brief + LangGraph Approval Workflow
  │
  ├─ ★ 시연 시나리오 1 — Drift 전체 Loop (v4.4 확정):
  │    FDC 유입 → A(예측) + B(감지) 병렬
  │    → C(Nelson 판정 + TTTM 기록) → E fast-path(심각도 평가, wafer HOLD)
  │    → 리포트 3종 병렬 탐색: 옵션① Recipe 튜닝 + 옵션② 실력치 재설정 + 옵션③ 정비안
  │    → E slow-path(RAG + Evidence Card + 순위 Brief) 비동기 합류
  │    → Approval Gate → 엔지니어 선택 → 실력치 반영(SPC 한계 갱신)
  │    + wafer RELEASE → 피드백 축적
  │
  └─ ★ 시연 시나리오 2 — PM 후 Qual 검증 + AE 재학습 (v4.4 확정):
       PM 완료 → Qual Wafer 투입 → A/B/C가 Qual 결과 산출
       → Dashboard 검증 요약 (AE Score, TTTM 갭, Nelson 위반, C65)
       → 엔지니어 승인 → AE 재학습 + 실력치 재산정 → 설비 정상 복귀
       ※ CT①(LightGBM 자동 재학습)은 시연 범위 외

  └─ ★ 시연 시나리오 3 — Recipe R2R Closed Loop (v4.5 신규):
       공정 조건 이탈 시나리오 주입 (센서 정상 범위 내, 조합 이동)
       → SHAP·AE가 포착 → Incident → Agent 옵션 3종 중 Recipe Tuning 1순위
       → 엔지니어 승인 (수정 승인 포함) → fdc.correction(recipe) 발행
       → 시뮬레이터가 해당 챔버 setpoint 반영 → 분포 정상화 + 예측 C65 하락 확인
       ※ 현업은 이 루프가 자동(레거시 DB API) — 우리는 승인 게이트로 감사 추적 확보
```

---

## 3. 기대 효과 및 비즈니스 임팩트

### 정량적 기대 효과

| 지표 | As-Is | To-Be | 기대 개선 |
|------|-------|-------|-----------|
| **가성 알람 비율** | 가성 86% (FDC Interlock, SNU 2020) | AI-SPC 1차 필터링 + **실력치 능동 보정** → 가성알람 40% 이상 감소 (AI-SPC, IJSRM 2025) | 엔지니어 확인 부담 대폭 절감 |
| **PM 후 모델 복구 시간** | 수동 재학습 + 수동 한계 재설정 3~5일 | Qual 검증 + 엔지니어 승인 → AE 재학습 + 실력치 재산정 완료까지 수 시간 이내 *(v4.4)* | Chamber Downtime 대폭 감소 |
| **Blind Attack 기간** | Fab-Out까지 2개월 | SPC 조기 경보 + Proxy Metrology → 조기 감지 | 미감지 excursion 건당 $21M 리스크(KLA 2014) 대폭 감소 |
| **이상 대응 시간** | 엔지니어 수동 과거 사례 검색 30분+ | Agent RAG 자동 검색 + Brief 생성 → 1분 이내 | 의사결정 속도 30× 향상 |
| **조치 신뢰도** | 엔지니어 개인 경험 의존 | 과거 성공률 + Evidence Card 기반 정량적 근거 | 승인 의사결정 품질 향상 |

### 기술적 의의

- **FDC → SPC → 실력치 보정 → Agent Closed Loop**의 완전한 오픈소스 레퍼런스
- **Tool-Using AI Agent** — "말만 하는 AI"가 아니라 제조 데이터 파이프라인 위에서 행동하는 AI
- **RAG 기반 Manufacturing Knowledge Layer** — 단순 매뉴얼 검색을 넘어 5타입 지식베이스 통합 검색
- **Human-in-the-Loop** — "모든 걸 자동화"가 아닌 "엔지니어의 판단을 데이터로 지원"
- **Approval Log as Knowledge Asset** — 승인/반려/수정 이력 자체가 Agent 학습 데이터가 되는 선순환 구조

---

## 4. Success Metrics

### Tier 1 — 기술 완성도

| # | 지표 | 측정 방법 | Threshold |
|---|------|----------|-----------|
| M1 | C65 예측 정확도 | RMSE + **R²** (valid_X 기준 — 현업은 R² 중시, v4.5) | v1 Baseline(RMSE 76.12) 대비 10% 이상 개선. ⚠️ R² 비정상 고평가 시 누수 검증 선행 |
| M2 | PM Drift 감지 속도 | PM 후 Drift Score Threshold 초과 Wafer 수 | 10 Wafer 이내 |
| M3 | Nelson Rules 평가 성능 | N1~N8 전체 구현 + Latency | 8/8 커버리지, < 1초 |
| M4 | TTTM 비교 정확도 | Chamber 간 Scale 차이 감지 | Reference 대비 ±3% 이상 차이 감지 |
| M5 | 실력치 재설정 적절성 *(v4.3)* | 재설정 후 섀도 평가 구간 가성알람 감소율 + 미탐 발생 여부 | 가성알람 35% 이상 감소 & 미탐 0건 |
| M6 | RAG 검색 정확도 | 유사 Case Top-3 Relevance | Similarity > 0.7인 결과 포함 |
| M7 | Agent Brief 완성도 | 정보 항목 포함율 | Rule, SHAP, 과거 성공률, 추천 조치 4/4 포함 |
| M8 | E2E Latency | FDC → Dashboard Alert | < 5초 (스트리밍), < 5분 (CT) |
| M12 | 라우팅 커버리지 *(v4.3)* | 신호 조합별 정의된 동작 존재 여부 | 100% (undefined behavior 0건) |
| M13 | Incident 그룹핑 *(v4.3)* | Alarm Storm 시나리오에서 승인 요청 수 | 동일 사건 중복 승인 요청 0건 |

### Tier 2 — 비즈니스 임팩트 (시뮬레이션 기반)

| # | 지표 | Threshold |
|---|------|-----------|
| M9 | 가성 알람 감소율 | 35% 이상 감소 |
| M10 | PM 복구 시간 단축 | 데모에서 **Qual 투입 → 검증 결과 제시 → 엔지니어 승인 → AE 재학습 + 실력치 재산정 완료**까지 5분 이내 *(v4.4)* |
| M11 | Agent 의사결정 지원 품질 | Evidence Card + Approval Brief 포함 |

> **측정 방법 (v4.4)**: M2·M5·M8·M12·M13 및 Qual 오탐/미탐율은 시뮬레이터 **Scorecard**(주입 정답 `sim_events` vs 시스템 판정 자동 대조 — `docs/시뮬레이터_스펙_v1.md` 7절)로 정량 측정한다.

### 성공 판정

- **Green**: Tier 1 9/10 달성 + Tier 2 2/3 달성
- **Yellow**: Tier 1 7/10 달성 + Tier 2 1/3 달성
- **Red**: Tier 1 6/10 미만 또는 M1(C65 예측) 미달 또는 M12(라우팅 커버리지) 미달

---

## 5. Risk & Assumptions

### 핵심 가정

| # | 가정 | 깨졌을 때 대안 |
|---|------|---------------|
| A1 | Etch 데이터의 센서 패턴(가변 길이 Trace, Step별 통계)을 추출할 수 있다 | Feature Engineering 접근 전환 |
| A2 | ~~PM Flag(C33)가 데이터에 존재한다~~ **확인 완료** — C33은 PM 경과 카운터이며 train 내 리셋(PM) 1회 관측 | (해소) |
| A3 | **로컬 LLM(AWS 셀프호스팅)**이 Agent 시나리오의 자연어 추론·JSON 생성에 충분하다 *(2026-07-06 로컬 확정)* | 모델 교체(D9 대안 2종) 또는 Rule-Based Fallback (템플릿 메시지)로 전환 |
| A4 | 데이터셋에서 C65(타겟)를 즉시 알 수 있다 — 실제 현장은 검사 후 수 주 지연 *(v4.3 명시)* | 재학습 데이터 구성 시 라벨 지연 시뮬레이션 추가 |
| A5 | 분포 이동의 "정상/열화" 판정에 필요한 근거(PM Flag, TTTM, 과거 사례)가 확보된다 | 판정 불확실 시 보수적 기본값 = 정비 점검 우선 |

### 핵심 위험

| # | 위험 | 가능성 | 영향도 | 완화 전략 |
|---|------|--------|--------|----------|
| R1 | 과적합 — 단일 Chamber/Recipe 데이터로 일반화 실패 | Medium | High | 교차 검증, Regularization 강화 |
| R2 | PM Drift Threshold 설정 실패 | Medium | High | ROC Curve 기반 최적 Threshold 도출 |
| R3 | SPC/보정/Agent 통합 복잡도 과다 | Medium | High | JSON Contract 사전 고정 후 병렬 개발 |
| R4 | RAG 검색 품질 부족 | Medium | Medium | 임베딩 모델 교체, 검색 결과 Reranking 추가 |
| R5 | ~~LLM API 비용/할당량~~ → **GPU 인스턴스 비용·VRAM 부족** *(로컬 확정으로 위험 성격 변경)* | Low | Medium | Agent 호출은 Approval Gate에서만 발동, 유사 응답 Cache, 양자화(AWQ/FP8)·인스턴스 조정(D20~D21) |
| R6 | **실력치 과다 상향 → 진짜 이상 미탐(true alarm masking)** *(v4.3 신규)* | Medium | High | 1회 재설정 폭 상한, 섀도 평가 통과 후 반영, 승인 게이트 필수, 미탐 모니터링 지표(M5) |

---

## 6. 오픈 이슈 & 운영 정책 *(v4.3 신규)*

### 6-1. ~~EWMA λ 장비별 차등~~ → **해소 (멘토 확인 완료, 2026-07-03)**

- 현업 재산정 산식은 EWMA가 아니라 **이전 3개월 rolling mean ± 3σ 자동 리캘리브레이션** — λ 개념 자체가 없어 "장비별 λ 차등" 질문 소멸 (4-1절 2트랙 구조 참조)
- 실력치 baseline의 Chamber × Recipe × Step 단위 분리는 유지 (멘토 확인)
- 잔여 결정: rolling window 표본 수 N — "3개월"의 시뮬 스케일 환산 (config A4)
- ~~선행 확인 필요: 실데이터 내 Equipment 실제 개수~~ **확인 완료**: Chamber 1대(C24)·Recipe 2종(C6) — 챔버 간 편차의 실데이터 근거 없음 → **TTTM·fleet 참조는 시뮬레이터 전담**. 대신 Recipe 축 분리(실력치 Chamber×Recipe×Step)는 실데이터로 실증 가능

### 6-2. 모델 재학습 주기 — 하이브리드 트리거 + 가드레일

| 구분 | 정책 |
|------|------|
| 트리거 (이벤트) | PM 카운터(C33) 리셋 감지 |
| 트리거 (Drift) | Drift Score threshold 초과 |
| 트리거 (성능) | valid RMSE 열화 감지 |
| 트리거 (정기) | 안전망 — 월 1회 |
| 가드레일 1 | **최소 데이터 조건**: PM 후 N wafer 축적 시에만 재학습 (Seasoning 기간 데이터 제외) |
| 가드레일 2 | **쿨다운**: 재학습 간 최소 간격 — drift 오탐으로 인한 재학습 폭주 방지 |
| 가드레일 3 | **재학습 ≠ 자동 배포**: champion–challenger 평가 통과 시에만 교체, MLflow 버전 + CHANGELOG 기록 |
| 가드레일 4 | **라벨 지연 가정 명시**: 현장에서는 C65 확보까지 수 주 소요 (A4 참조) |
| 미결정 *(v4.4)* | **CT①(LightGBM) 재학습 트리거 시점**: 실측마다 / 일배치 / 월배치 중 Phase 4에서 확정 — 데이터 축적 속도·재학습 비용·열화 속도 기준 |

### 6-3. 운영 견고성

- **Consumer 멱등 처리**: 재시작 시 같은 alert 재소비로 Incident 중복 생성 금지 (incident_id 기준 dedup)
- **파이프라인 Health 모니터링**: 감시 시스템 자체의 생존 확인 (heartbeat) — 감시가 죽은 걸 아무도 모르는 상황 방지
- 메시지 역직렬화 실패 시 스킵 + 로그 (헌법 6-2)
- **데이터 보존 정책** *(v4.3)*: 원시 Trace는 N개월 후 다운샘플(TimescaleDB retention/continuous aggregate), 종료 Incident는 아카이브 테이블로 이관. Evidence·approval_records는 영구 보존 (지식 자산)

### 6-4. 기타

- Reference Chamber 재검증 주기 (3-2절 참조)
- Pending 승인 건 타임아웃/에스컬레이션 정책 (6-5절 참조)
- ~~가성알람 감소율 측정용 ground truth 설계~~ **설계 완료** *(v4.4)*: 시뮬레이터 `sim_events`(진성/가성 라벨 내장) + Scorecard 자동 채점 — `docs/시뮬레이터_스펙_v1.md`
- ~~실데이터 내 LOT 매핑 컬럼 존재 여부 확인~~ **확인 완료**: C20(Lot ID) + C34(FOUP 슬롯 1~25) — LOT HOLD 파생 구현 가능

---

## 7. Competitive Differentiation

### 포지셔닝

> **우리는 상용 솔루션과 경쟁하는 것이 아니라, 상용 솔루션이 채우지 못하는 "설명 가능하고 오픈된 Agentic Closed Loop 레퍼런스"라는 빈 공간을 점유한다.**

### 상용 솔루션 대비 차별점

| 기능 영역 | Applied Materials | TensorBlue | 우리 |
|-----------|-------------------|------------|------|
| FDC (Fault Detection) | ✅ | ✅ | ✅ |
| SPC (Statistical) | ✅ | ✅ | ✅ |
| R2R / APC (Recipe 제어) | ✅ (자동) | ✅ (자동) | ✅ **HITL R2R** — SHAP 근거 + 승인 게이트 (v4.5 부활, 현업은 자동·우리는 설명가능+승인) |
| FDC 실력치 능동 보정 제안 | △ | - | ✅ |
| RAG Knowledge Layer | - | - | ✅ |
| Tool-Using AI Agent | △ | - | ✅ |
| Evidence Card 기반 근거 제시 | - | - | ✅ |
| Human-in-the-Loop AI | △ | - | ✅ |
| Approval Log → Agent 학습 | - | - | ✅ |
| Open Source | - | - | ✅ |

### 핵심 차별화 3가지

1. **RAG 기반 Manufacturing Knowledge Layer**: 상용 솔루션은 Rule-Based Alert(임계값 초과 → Email)이거나 완전 자동화된 APC. 과거 유사 사례·매뉴얼·실력치 변경 이력을 RAG로 자동 검색하여 Evidence Card로 제시하는 시스템은 시장에 없다.

2. **Tool-Using AI Agent**: 단순 LLM 챗봇이 아니라, 제조 데이터 파이프라인의 API를 직접 호출하여 SPC 상태 조회·TTTM 비교·RAG 검색·실력치 시뮬레이션을 수행하는 Agent. "말만 하는 AI"가 아니라 "행동하는 AI".

3. **Approval Log as Knowledge Asset**: 승인/반려/수정 이력이 축적되면서 Agent의 추천 품질이 지속적으로 개선되는 선순환 구조. 어떤 추천이 승인되었는가, 엔지니어는 무엇을 수정했는가, 반려 사유는 무엇인가, 승인 후 실제 개선되었는가를 학습.

---

## 최종 정리

> **반도체 Etch 공정의 FDC Trace 데이터를 기반으로 C65 불량을 예측하고, SPC/TTTM으로 이상 원인을 분석하며, AI Agent가 Recipe Tuning(R2R)·실력치 재설정·정비 조치안을 산출해 엔지니어 승인 후 적용한다. 이상 wafer는 HOLD → 스크랩/해제 Disposition으로 처리한다. 이후 RAG 기반 AI Agent가 과거 유사 사례·공정 매뉴얼·실력치 변경 이력을 검색하여 엔지니어 승인용 근거 리포트를 생성하는 Human-in-the-Loop 제조 AI 플랫폼을 구축한다.**

이 시나리오는 **인터락 → 분석 → 근거 검색 → 실력치/정비 조치 제안 → 승인 → 조치 저장 → 재학습**까지 보여주는 Agentic Closed Loop이다.

---

### 기술 스택 요약

| 영역 | 기술 |
|------|------|
| Streaming | Kafka (6 Topic) |
| API | FastAPI |
| Time-series DB | PostgreSQL / TimescaleDB |
| Vector DB | Qdrant (payload 필터 + 벡터 검색) |
| ML | LightGBM + SHAP |
| DL | PyTorch Autoencoder |
| Workflow | Airflow |
| Agent Framework | LangGraph (Agent + 승인 상태머신) |
| Dashboard | **React** (SPA, FastAPI REST + WebSocket/SSE) *(v4.4: Streamlit → React 전환)* |
| Container | Docker Compose |
| MLOps | MLflow + Optuna |

---

*SK hynix · M&T Data Science · ASAC DE 2기 · Ver 4.5*

---

## 부록. 현업 용어 매핑 *(v4.5)*

| 현업 용어 (멘토) | 본 시스템 대응 |
|---|---|
| R2R (Run-to-Run) | 넓은 의미의 보정 루프 — 멘토는 "FDC 실력치"도 R2R로 지칭. 본 문서는 ① Recipe R2R(공정), ② Model R2R(모델 bias), ③ 실력치 리캘리브레이션(기준선)으로 구분 |
| RTD (Real-Time Dispatcher) | 자동 inhibit (챔버 배차 제외) |
| BM (Breakdown Maintenance) | 정비 조치 (Maintenance Agent 권고 → 승인) |
| Sample wafer | Qual wafer (QUAL 모드 5장) |
| crazy wafer | spot성 단발 outlier — 자동 HOLD → 승인 스크랩 |
| WT (Wafer Test) | 최종 검사 — C65(fail bit count)의 출처 |
| RF time | C33 — RF 누적 시간, 리셋 = PM (PM Cycle 식별 기준) |
| Cycle | PM ~ 다음 PM 구간 — CT① 재학습 단위 |
