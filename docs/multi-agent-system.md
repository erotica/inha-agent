# 🏥 인하대병원 Multi-Agent 시스템 — 상세 설계

> 20개 병원 AI 프로젝트를 하나의 통합 **Multi-Agent 시스템**으로 운영하는 아키텍처

---

## 1. 시스템 개요

### 1.1 왜 Multi-Agent인가?

| 단일 모델 접근법 | Multi-Agent 접근법 |
|:----------------|:-------------------|
| 각 프로젝트가 독립적 실행 | Agent 간 **데이터/예측 공유** |
| 중복 데이터 수집 | **공유 Knowledge Layer** |
| 결과물 단일 출력 | **협업 추론** (예: 병상배치 + 퇴원예측) |
| 수동 통합 필요 | **자동 오케스트레이션** |

### 1.2 핵심 원칙

```
공유 데이터 레이어 (EMR/OCS/LIS)
        ↓
 전담 Agent 들 (20개 프로젝트)
        ↓
 오케스트레이터 (협업 + 충돌 해결)
        ↓
 통합 대시보드 + Action Engine
```

---

## 2. Agent 구성

### 2.1 Agent 목록 (20개 → 8개 그룹)

```
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator Agent                        │
│  (전체 조정, 우선순위 결정, 자원 배분)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Patient     │  │ Resource    │  │ Clinical    │         │
│  │ Flow Agent  │  │ Agent       │  │ Quality     │         │
│  │             │  │             │  │ Agent       │         │
│  │• 입원예측   │  │• 수술실     │  │• 처방검토   │         │
│  │• 응급실    │  │• 병상       │  │• 약품상호   │         │
│  │• 퇴원예측  │  │• 장비정비   │  │• 감염탐지   │         │
│  │• 재입원    │  │• 검체이송   │  │• 기록요약   │         │
│  │• No-Show   │  │• 간호스케줄 │  │• 처방패턴   │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                  │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐         │
│  │ Financial   │  │ Supply      │  │ Research    │         │
│  │ Agent       │  │ Chain Agent │  │ Agent       │         │
│  │             │  │             │  │             │         │
│  │• 진료비심사 │  │• 재고예측   │  │• 임상시험   │         │
│  │• 폐기물    │  │• 구매추천   │  │• 환자매칭   │         │
│  │• 만족도    │  │• 검사량예측 │  │• 문헌분석   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                    Knowledge Layer                           │
│  (통합 Feature Store + Model Registry + Event Bus)          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Agent별 상세

#### 🧑‍⚕️ Patient Flow Agent (환자 흐름)
| 담당 프로젝트 | Agent Role | 핵심 협업 대상 |
|:-------------|:-----------|:--------------|
| 1. 병상 배치 | 입원 수요 예측 → 병상 할당 | Resource Agent |
| 2. 응급실 도착 | ED 도착 예측 → 의료진 요청 | Resource Agent |
| 6. 퇴원 예측 | 퇴원 시점 예측 → 병상 회전 | Resource Agent |
| 9. No-Show | No-Show 위험 예측 → 오버부킹 | Financial Agent |
| 13. 재입원 | 재입원 위험 평가 → 퇴원 관리 | Clinical Quality Agent |

**Agent 행동 예시:**
```python
class PatientFlowAgent:
    """환자 흐름 관리 Agent"""
    
    def predict_bed_demand(self, date) -> dict:
        """입원 수요 예측 및 병상 부족 시 알림"""
        admissions = self.predict_admissions(date)
        discharges = self.call_agent('resource', 'get_planned_discharges', date)
        bed_gap = admissions - discharges
        if bed_gap > 0:
            self.alert(f"🛏️ {bed_gap}개 병상 부족 예상")
            self.call_agent('resource', 'suggest_bed_reallocation', bed_gap)
        return {'admissions': admissions, 'discharges': discharges}
    
    def on_event(self, event: Event):
        """실시간 이벤트 처리"""
        if event.type == 'ED_ARRIVAL_SURGE':
            self.call_agent('resource', 'request_extra_staff', 
                          dept='ED', count=event.data['count'])
```

#### 🏗️ Resource Agent (자원 관리)
| 담당 프로젝트 | Agent Role | 핵심 협업 대상 |
|:-------------|:-----------|:--------------|
| 3. 수술실 스케줄 | 수술실 배정 최적화 | Patient Flow |
| 14. 검체 이송 | 로봇 경로 최적화 | Supply Chain |
| 16. 간호사 근무표 | 인력 배치 최적화 | Patient Flow |
| 19. 장비 예측 정비 | 장비 가용성 관리 | 모든 Agent |

#### 🩺 Clinical Quality Agent (임상 질 관리)
| 담당 프로젝트 | Agent Role | 핵심 협업 대상 |
|:-------------|:-----------|:--------------|
| 4. 처방 검토 | 처방 안전성 검증 | 약품 상호작용 Agent |
| 7. 감염 탐지 | 감염 조기 경보 | Patient Flow |
| 8. 기록 요약 | 의무 기록 자동화 | 모든 Agent |
| 11. 약품 상호작용 | 약물 안전 관리 | 처방 검토 Agent |
| 18. 처방 패턴 | 진료 행태 분석 | Quality Agent |

---

## 3. 통신 및 협업 프로토콜

### 3.1 Event Bus 아키텍처

```python
# agent_framework/event_bus.py
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Callable
import asyncio

class EventType(Enum):
    PATIENT_ADMITTED = "patient_admitted"
    PATIENT_DISCHARGED = "patient_discharged"
    ED_ARRIVAL_SURGE = "ed_arrival_surge"
    BED_SHORTAGE = "bed_shortage"
    STAFF_REQUEST = "staff_request"
    SUPPLY_LOW = "supply_low"
    INFECTION_ALERT = "infection_alert"
    PREDICTION_UPDATED = "prediction_updated"
    ANOMALY_DETECTED = "anomaly_detected"

@dataclass
class Event:
    type: EventType
    source: str  # agent_id
    data: dict
    priority: int = 0  # 0=normal, 1=high, 2=critical
    timestamp: float = field(default_factory=time.time)

class EventBus:
    """중앙 이벤트 버스 — Agent 간 통신"""
    
    def __init__(self):
        self.subscribers: dict[EventType, list[Callable]] = {}
        self.event_log: list[Event] = []
    
    def subscribe(self, event_type: EventType, callback: Callable):
        """이벤트 구독"""
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(callback)
    
    async def emit(self, event: Event):
        """이벤트 발행"""
        self.event_log.append(event)
        for callback in self.subscribers.get(event.type, []):
            await callback(event)
    
    def get_recent_events(self, minutes: int = 60) -> list:
        """최근 이벤트 조회"""
        cutoff = time.time() - minutes * 60
        return [e for e in self.event_log if e.timestamp > cutoff]
```

### 3.2 Agent 간 협업 시나리오

#### 시나리오: 수술실 과밀 → 전파

```
수술실 Agent: "내일 오전 3개실 부족 예상"
     ↓ (Event: OR_OVERBOOKED)
Resource Agent: "수술 연기 가능 목록 생성 중..."
     ↓
Patient Flow Agent: "연기된 수술 환자 → 다음 주 재예약 필요"
     ↓
Supply Chain Agent: "연기된 수술 재고 → 타 수술실로 재배치"
     ↓
Orchestrator: 최종 결정 → 관련 부서 알림
```

#### 시나리오: 감염 의심 → 통합 대응

```
Clinical Agent: "3병동 발열 환자 5명 → 감염 의심"
     ↓ (Event: INFECTION_ALERT)
Patient Flow Agent: "3병동 신규 입원 중단 권고"
Resource Agent: "격리실 가동률 80% → 2개 추가 준비"
Supply Chain Agent: "방역 물품 재고 확인 → 부족 시 발주"
     ↓
Orchestrator: "감염관리위원회 소집 필요 → 자동 보고서 생성"
```

---

## 4. 시스템 아키텍처

### 4.1 기술 스택

| 계층 | 기술 | 용도 |
|:-----|:------|:------|
| **Agent Framework** | LangChain / CrewAI / Custom | Agent 오케스트레이션 |
| **LLM Backend** | Claude Code CLI + Ollama | Agent 추론/결정 |
| **Data Layer** | PostgreSQL + Redis | 예측 데이터 + 캐싱 |
| **Event Bus** | Redis Pub/Sub + Kafka | 실시간 이벤트 |
| **Feature Store** | Feast / Custom | 통합 Feature 관리 |
| **Model Registry** | MLflow | 모델 버전 관리 |
| **API Gateway** | FastAPI + GraphQL | 통합 API |
| **Dashboard** | React + D3.js + Grafana | 시각화 |
| **Infra** | Docker + Kubernetes | 확장성 |

### 4.2 디렉토리 구조

```
hospital-agent/
├── agent_framework/
│   ├── base_agent.py        # Agent 기본 클래스
│   ├── event_bus.py         # 이벤트 버스
│   ├── orchestrator.py      # 오케스트레이터
│   ├── knowledge_graph.py   # 지식 그래프
│   └── message_queue.py     # 메시지 큐
├── agents/
│   ├── patient_flow/        # 환자 흐름 Agent
│   ├── resource/            # 자원 관리 Agent
│   ├── clinical_quality/    # 임상 질 Agent
│   ├── financial/           # 재무 Agent
│   ├── supply_chain/        # 공급망 Agent
│   └── research/            # 연구 Agent
├── models/
│   ├── registry/            # 모델 저장소
│   └── predictors/          # 개별 예측 모델
├── shared/
│   ├── feature_store/       # 통합 Feature Store
│   ├── data_layer/          # EMR/OCS/LIS 연동
│   └── schemas/             # 공통 데이터 스키마
├── frontend/
│   ├── dashboard/           # 통합 대시보드
│   └── agent_control/       # Agent 제어판
├── api/
│   └── gateway.py           # 통합 API Gateway
├── docker-compose.yml
└── kubernetes/
    └── agent-deployment.yaml
```

### 4.3 Agent Base Class

```python
# agent_framework/base_agent.py

class BaseAgent(ABC):
    """모든 Agent의 기본 클래스"""
    
    def __init__(self, agent_id: str, name: str, 
                 llm_backend: str = "claude"):
        self.agent_id = agent_id
        self.name = name
        self.event_bus = EventBus()
        self.memory = AgentMemory()
        self.llm = self._init_llm(llm_backend)
        self.projects: list[str] = []  # 담당 프로젝트 IDs
        self.status = AgentStatus.IDLE
    
    @abstractmethod
    async def analyze(self, context: dict) -> dict:
        """주기적 분석 실행"""
        pass
    
    @abstractmethod
    async def handle_event(self, event: Event):
        """이벤트 처리"""
        pass
    
    async def call_agent(self, target: str, action: str, **kwargs):
        """다른 Agent 호출 (RPC)"""
        return await self.event_bus.rpc_call(target, action, kwargs)
    
    def log(self, message: str, level: str = "INFO"):
        """Agent 로깅"""
        logger.info(f"[{self.agent_id}] {message}")
    
    def get_metrics(self) -> dict:
        """Agent 상태 메트릭"""
        return {
            'agent_id': self.agent_id,
            'status': self.status.value,
            'projects': self.projects,
            'predictions_today': self.memory.predictions_today,
            'events_processed': len(self.memory.event_history),
        }
```

### 4.4 Orchestrator

```python
# agent_framework/orchestrator.py

class Orchestrator:
    """Multi-Agent 오케스트레이터"""
    
    def __init__(self):
        self.agents: dict[str, BaseAgent] = {}
        self.event_bus = EventBus()
        self.scheduler = BackgroundScheduler()
        self.conflict_resolver = ConflictResolver()
    
    def register_agent(self, agent: BaseAgent):
        """Agent 등록"""
        self.agents[agent.agent_id] = agent
        self.log(f"✅ Agent 등록: {agent.name} ({agent.agent_id})")
    
    async def run_daily_cycle(self):
        """일일 자동 실행 사이클"""
        self.log("🔄 일일 Agent 사이클 시작")
        
        # 1. 모든 Agent 분석 실행
        results = {}
        for agent in self.agents.values():
            results[agent.agent_id] = await agent.analyze({
                'date': today(),
                'previous_results': results
            })
        
        # 2. 충돌 감지 및 해결
        conflicts = self.detect_conflicts(results)
        for conflict in conflicts:
            resolution = await self.conflict_resolver.resolve(conflict)
            self.log(f"⚖️ 충돌 해결: {conflict.description} → {resolution}")
        
        # 3. 통합 보고서 생성
        report = await self.generate_integrated_report(results)
        
        # 4. 액션 실행
        actions = self.prioritize_actions(report['actions'])
        for action in actions[:5]:  # 상위 5개만 실행
            await self.execute_action(action)
        
        self.log("✅ 일일 사이클 완료")
        return report
    
    def detect_conflicts(self, results: dict) -> list:
        """Agent 간 추천 충돌 감지"""
        conflicts = []
        # 예: 수술실 Agent vs 병상 Agent 충돌
        or_recommendation = results.get('resource', {}).get('or_schedule', {})
        bed_recommendation = results.get('patient_flow', {}).get('bed_plan', {})
        # ... 충돌 감지 로직
        return conflicts
```

---

## 5. 데이터 흐름

### 5.1 통합 Feature Store

```python
# shared/feature_store/feature_registry.py

class FeatureRegistry:
    """전 Agent 공유 Feature Store"""
    
    FEATURES = {
        'patient_count': {
            'description': '현재 입원 환자 수',
            'source': 'EMR',
            'update_freq': 'realtime',
            'owners': ['patient_flow', 'resource'],
        },
        'bed_occupancy_rate': {
            'description': '병상 가동률',
            'source': 'EMR',
            'update_freq': 'hourly',
            'owners': ['patient_flow', 'resource'],
        },
        'ed_waiting_count': {
            'description': '응급실 대기 환자 수',
            'source': 'ED System',
            'update_freq': 'realtime',
            'owners': ['patient_flow'],
        },
        'or_utilization': {
            'description': '수술실 가동률',
            'source': 'OR System',
            'update_freq': 'daily',
            'owners': ['resource'],
        },
        'inventory_alert_count': {
            'description': '재고 부족 경보 수',
            'source': 'Inventory System',
            'update_freq': 'realtime',
            'owners': ['supply_chain'],
        },
        'infection_warnings': {
            'description': '감염 의심 건수',
            'source': 'LIS/EMR',
            'update_freq': 'realtime',
            'owners': ['clinical_quality'],
        },
        'claim_error_rate': {
            'description': '진료비 청구 오류율',
            'source': 'Billing System',
            'update_freq': 'daily',
            'owners': ['financial'],
        },
    }
    
    def get(self, feature_name: str) -> Any:
        """Feature 조회"""
        pass
    
    def subscribe(self, feature_name: str, callback: Callable):
        """Feature 변경 구독"""
        pass
```

### 5.2 데이터 수집기 (Data Collector)

```python
# shared/data_layer/collector.py

class HospitalDataCollector:
    """병원 데이터 통합 수집기"""
    
    SOURCES = {
        'emr': EMRConnector(db_type='oracle'),
        'lis': LISConnector(db_type='mssql'),
        'ocs': OCSConnector(db_type='oracle'),
        'nurse': NurseStationAPI(endpoint='...'),
        'inventory': InventoryDB(),
        'billing': BillingSystemAPI(),
        'ed': EDSystemConnector(),
    }
    
    async def collect_all(self) -> dict:
        """전체 데이터 수집"""
        data = {}
        for name, source in self.SOURCES.items():
            try:
                data[name] = await source.fetch()
            except Exception as e:
                self.log(f"❌ {name} 수집 실패: {e}")
        return data
```

---

## 6. 배포 아키텍처

### 6.1 컨테이너 구성

```yaml
# docker-compose.yml
version: '3.8'
services:
  orchestrator:
    build: .
    command: python -m agent_framework.orchestrator
  
  agent-patient-flow:
    build: .
    command: python -m agents.patient_flow.main
  
  agent-resource:
    build: .
    command: python -m agents.resource.main
  
  agent-clinical:
    build: .
    command: python -m agents.clinical_quality.main
  
  agent-financial:
    build: .
    command: python -m agents.financial.main
  
  agent-supply-chain:
    build: .
    command: python -m agents.supply_chain.main
  
  agent-research:
    build: .
    command: python -m agents.research.main
  
  event-bus:
    image: redis:7-alpine
  
  feature-store:
    image: postgres:15
  
  dashboard:
    build: frontend
    ports: ["3000:3000"]
  
  api-gateway:
    build: .
    command: uvicorn api.gateway:app --host 0.0.0.0 --port 8000
    ports: ["8000:8000"]
```

### 6.2 Agent 간 통신

```
┌──────────┐     RPC Calls     ┌──────────┐
│ Agent A  │◄─────────────────►│ Agent B  │
└──────────┘                   └──────────┘
     │                              │
     │     Event Bus (Redis)        │
     ├──────────────────────────────┤
     │     Feature Store (PG)        │
     ├──────────────────────────────┤
     │     Model Registry (MLflow)   │
     └──────────────────────────────┘
```

---

## 7. 구현 로드맵

| Phase | 기간 | 내용 | 산출물 |
|:------|:----:|:------|:--------|
| **P0** | 2주 | Agent Framework + Event Bus | Base Agent, 통신 인프라 |
| **P1** | 4주 | Patient Flow + Resource Agent | 2개 Agent + Feature Store |
| **P2** | 4주 | Clinical + Supply Chain Agent | 4개 Agent |
| **P3** | 3주 | Financial + Research Agent | 6개 Agent 완료 |
| **P4** | 3주 | Orchestrator + Dashboard | 통합 대시보드 + API |
| **P5** | 2주 | Kubernetes 배포 + 모니터링 | 운영 환경 |

**총 예상 기간: 18주**

---

## 8. 기대 효과

| 항목 | 단일 프로젝트 | Multi-Agent 통합 |
|:-----|:------------:|:----------------:|
| 데이터 중복 | ⚠️ 각 프로젝트 별도 수집 | ✅ **Feature Store 1회 수집** |
| 예측 정확도 | 70-80% | ✅ **85-95%** (상호 보완) |
| 의사결정 속도 | 수동 통합 | ✅ **실시간 자동 결정** |
| 확장성 | 프로젝트 추가 시 재구축 | ✅ **Agent만 추가 등록** |
| 유지보수 | 20개 시스템 개별 관리 | ✅ **통합 플랫폼** |
| 총 예상 비용 | 각 2-5천만원 | ✅ **1억원 (통합 60% 절감)** |

---

> 📍 **저장소:** inhwa-agent 프로젝트에 통합 예정
