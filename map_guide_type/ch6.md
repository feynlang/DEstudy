# Ch6. Data flow design Pattern

<aside>
- key takeaway? 

    - 전체 디자인 패턴을 조망하는 동시에 적합한 문제 상황과 발생가능한 결과와 예외 처리 등을 다루고 있는 책. 따라서 케이스 스터디에 알맞은 책으로 보이지만(적용하면 좋았을 사례와 책에서 다루고 있지 않은 또 다른 예외 상황들 등) 
    - 나는 아직 경험한 실무 케이스가 없어서 추후에 진행하게 될 프로젝트나 ETL 프로세스 등에 적용할 수 있는 패턴들의 큰 밑그림을 그려놓는 식으로 접근하는게 맞아보임.

</aside>

## Pattern Map

| 분류 | 패턴 | 1. problem | 2. solution | 3. consequence | 4. example |
| --- | --- | --- | --- | --- | --- |
| Sequence | **Local Sequencer

같은 파이프라인/같은 잡 안에서 순차 실행** | 실패 시 처음부터 재실행, 코드 비대화, 디버깅 어려움 | 큰 작업을 작은 순차 단계로 분해 | [+] 비싼 단계 재실행 방지 가능
[-] 너무 잘게 쪼개면 스케줄링 부담 증가 |   1. Airflow: `>>`, 
  2. aws EMR: Step API(실패시 의존성 구성 필요) ← vs CRON(간결; 독립성x) |
|  | **Isolated Sequencer

서로 분리된 파이프라인 간 순차 의존** | 팀 간 책임 분리, 독립 파이프라인 협업 | 분리된 파이프라인 사이 의존성 연결 | [+] 팀별 분리 운영 가능
[-] 스케줄 정합성, 커뮤니케이션 비용 |   1. airflow(waiter컴포넌트, 
  • 암묵적 데이터(sensor) 의존성 
  • 2개 연산자marker&sensor 독립적 사용 |
| Fan-In | **Aligned Fan-In

여러 부모가 모두 성공→ 자식 실행** | 여러 부분 결과가 모두 필요함 | 병렬 브랜치 결과를 정확하게 합치기 | [+] 작은 병렬 작업으로 빠른 피드백
[-] 인프라 스파이크, skew, 스케줄링 오버헤드 |   • (여러 브랜치 + 공통 child)
  1. Airflow: for문 안 파이프라인 시퀀스 1번만 정→전용태스크 생성 후 공통 상위태스크에 연결
  2. SQL: `UNION`, `JOIN`
  3. Spark: `unionByName` |
|  | **Unaligned Fan-In

일부 부모 실패를 허용→ 자식 실행** | 일부 입력 누락 때문에 전체 결과가 막힘 | 실패 일부를 허용한 채 결과 생성 | [+] 전체 지연 감소
[-] partial 품질 고지 필요, 흐름 이해 어려움 |   • (올바른 트리거 설정 사용)
  1. Airflow: `trigger_rule=ALL_DONE`, 서브쿼리 활용~새로운 플래그(완전한 입력 데이터에 기반한것인지 유무 확인)
  2. aws: step function(덜 선언적인 구현)
detector(1회)
→processor(성공여부 플래그생성)
→creator(오류가 있다면 테이블에 메타데이터 주석) |
| Fan-Out | **Parallel Split

하나의 부모에서 여러 자식이 병렬 분기** | 같은 데이터를 두 군데 이상에 써야 함 | 공통 입력 기반 다중 후속 처리 | [+] 공통 입력 재사용, 다중 처리 효율
[-] 느린 브랜치가 병목, 하드웨어 불일치 문제 |   1. Airflow 공통 부모 후 여러 downstream
  2. Spark: `persist()` 후 다중 write
  3. 재시도 시 두번 쓰기방지(델타레이크: txnVersion, txnAddId) |
|  | **Exclusive Choice

하나의 부모에서 조건에 따라 한 갈래만 선택** | 특정 날짜 이후 새 로직, 이전 날짜는 구 로직 | 조건에 따라 경로 선택 | [+] 분기 남발 시 복잡도 폭증
[-] 필요 없는 경로 실행 방지 |   1. Airflow: BranchPythonOperator, if-else, switch
  2. Spark: factory pattern
if-else, switch, factory pattern |
| Orchestration | **Single Runner

같은 파이프라인은 항상 하나만 실행** | incremental/stateful 처리 | 순차성 보장 | [+] 논리적 정합성 보장
[-] 느린 처리, 높은 backfill 비용 |   1. Airflow:`max_active_runs=1`, `depends_on_past=True` 
  2. aws EMR: StepConcurrencyLevel 매개변수
 |
|  | **Concurrent Runner

같은 파이프라인을 여러 개 동시 실행** | 독립 실행인데 순차 실행 때문에 병목 발생 | 처리량과 지연 개선 | [+] 지연 완화, backfill 가속
[-] resource starvation(workload management),
shared state 충돌 |   1. Airflow: `max_active_runs>1` |

## Choice

| 질문(trade-off) | 관련 패턴 |
| --- | --- |
| 무엇이 하나의 실행 단위여야 하는가? | Local Sequencer, 
Fan-In/Fan-Out 전반 |
| 무엇을 독립적으로 재시작해야 하는가? | Local Sequencer, Aligned Fan-In |
| 부분 결과를 허용할 것인가? | Aligned Fan-In vs Unaligned Fan-In |
| 팀 경계 때문에 파이프라인을 분리해야 하는가? | Isolated Sequencer |
| 같은 입력을 여러 후속 처리에 써야 하는가? | Parallel Split |
| 여러 갈래 중 하나만 선택해야 하는가? | Exclusive Choice |
| 현재 실행이 이전 실행에 의존하는가? | Single Runner |
| 각 실행이 서로 독립적인가? | Concurrent Runner |

| 상황 | 패턴 |
| --- | --- |
| 한 잡이 너무 커서 실패 때마다 처음부터 다시 돈다 | Local Sequencer |
| 다른 팀 파이프라인과 연결해야 한다 | Isolated Sequencer |
| 24개 시간별 처리 후 하루 집계를 만든다 | Aligned Fan-In |
| 24개 중 몇 개 실패해도 일단 결과를 내야 한다 | Unaligned Fan-In |
| 같은 입력으로 CSV와 Delta를 동시에 써야 한다 | Parallel Split |
| 특정 날짜 전후로 다른 로직을 적용해야 한다 | Exclusive Choice |
| 이전 실행 결과를 다음 실행이 반드시 참조한다 | Single Runner |
| 각 실행이 서로 독립적이어서 동시에 돌아도 된다 | Concurrent Runner |