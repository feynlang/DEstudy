# Ch7. Data Security design Pattern

## Pattern Map

| 분류 | **패턴** | 1. problem | 2. solution | 3. consequence
 | 4. example |
| --- | --- | --- | --- | --- | --- |
| Data Removal | **Vertical Partitioner

개인정보 저장 구조의 분리계층을 만들어 저장** | 개인정보 삭제 요청 시 삭제 대상 데이터가 너무 많음 | mutable/immutable(또는 PII) 속성을 수직 분할 저장 | [+] 제거할 데이터 양 절감(=삭제 비용),
폴리글랏 지속성

[-] 읽기 시 join 필요, 조회 복잡도 증가(일종의 정규화) |   1. 아파치 스파크: foreachBatch(topic 분리 저장, merge작업으로 델타레이크 테이블로 변환o)
  2. Delta Lake: delete(보존기간 초과)
  3. 아파치 카프카: 툼스톤 메시지(제거-마커)→컴팩션 프로세스 |
|  | **In-Place Overwriter

기존 데이터를 삭제 대상만 제외하여 rewrite** | 레거시 구조에서 개인정보 삭제를 해야 함 | 전체 데이터셋을 읽고 삭제 대상 제외 후 다시 씀
(오픈테이블파일형식,
중간 스테이징 영역,
컴팩션 프로세스 이용) | [+] 범용성, 단순성

[-] I/O 오버헤드(→ 아파치 파케이, 관련없는 읽기작업skip), 
조회 비용(→제거 요청 그룹화, 1개 파이프라인) |   1. 아파치 스파크: 텍스트API, 필터 사용속성만 추출,
  2. aws: 최종출력으로 승격 |
| Access Control | **Fine-Grained Accessor for Tables

테이블 접근 권한을 행·열 수준까지 세분화** | 테이블은 접근 가능하지만 특정 열/행은 제한해야 함 | column-level, row-level access policy 적용 | [+] 데이터 내부까지 세밀 제어

[-] 적용범위 제한(연결세션 속성),
중첩구조 컬럼 제약,쿼리 오버헤드 |   1. PostgreSQL: grant
  2. aws dynamoDB: `dynamodb:LeadingKeys` |
|  | **Fine-Grained Accessor for Resources

클라우드 리소스 권한을 최소 권한 원칙으로 세밀하게 제한** | 클라우드 리소스 권한이 너무 넓음 | least privilege 기반 IAM/resource policy
(자원수준/ID수준) | [+] 최소 권한 원칙 구현

[-] 트레이드오프(유지 관리가 어려운 작은 정책),
시스템 복잡성, 
사용자 정의 정책 할당량 제한 |   1. AWS s3: id기반/자원기반 |
| Data Protection | **Encryptor

저장 중이거나(클라이언트측/서버측-CSP지원 추상화계층) 
전송 중인 데이터를 암호화** | 저장 중/전송 중 데이터 탈취 위험 | at rest / in transit 암호화 | [+] 원본 데이터 자체 보호

[-] CPU 오버헤드, 키 분실 리스크(→소프트삭제), 프로토콜 갱신(전송중 암호화—최신상태 유지) |   1. aws kms: 암호화 키 정의, 권한부여(필수 매개변수: deletion_window_in_days), s3버킷과 결합 |
|  | **Anonymizer

데이터셋의 개인 식별이 불가능하도록 민감 데이터를 제거·변형·합성데이터 대체** | 외부 공유 시 PII를 완전히 제거해야 함 | 제거/노이즈 추가/합성 데이터로 비식별화 | [+] 강한 비식별(법적 리스크↓)

[-] 정보 손실 큼, 분석 활용성 저하 |   1. 제거&합성데이터 대체(스파크API, drop & withColumn)
  2. →익명화(Faker파이썬 라이브러리) |
|  | **Pseudo-Anonymizer

민감 정보를 가명화** | 값을 숨기되 분석 의미는 일부 유지해야 함 | 마스킹/토큰화(매핑 정보 저장,보안)/해싱/암호화(복원가능) | [+] 보안과 분석의 밸런스(분석 활용성 유지)

[-] 결합 시 재식별 가능성, 정보유실/데이터타입 손실 |   1. 스파크: mapInPandas, 컬럼변환 |
| Connectivity | **Secrets Pointer

자격증명 값을 비밀 관리자 서비스를 활용해 저장소 참조로 대체** | 코드/Git에 비밀번호, API 키 저장 위험 | secret 값 대신 secret 참조 사용 | [+] 자격 증명 유출 방지 및 중앙 관리

[-] 캐시 무효화 문제, 로그에 포함 시 유출 위험, 비밀 컨슈머의 필요(human관리자 or IaC스택) |   1. 스파크: postgresql로 연결→json파일변환 |
|  | **Secretless Connector

자격증명 없이 IAM서비스나 인증서(CA) 기반으로 안전하게 연결** | 자격증명 자체를 관리하고 싶지 않음 | IAM/Service Account/Certificate 기반 연결 | [+] 관리 부담 zero

[-] 초기 구성 필요(역할 가정 권한), 인증서 회전 관리 오버헤드(생성,공유,기존/new 자격증명 모두 지원) |   1. 스파크: postgresql로 인증서 기반 연결 |

## Choice

| 상황 | 패턴 |
| --- | --- |
| 처음부터 삭제 친화적으로 설계할 수 있는 신규/재설계 프로젝트
(ex. 설계 단계부터 GDPR/CCPA 삭제 요청) | Vertical Partitioner |
| 기존 구조를 바꾸기 어려운 레거시 시스템
(ex. 이미 쌓인 대용량 레거시 데이터 삭제) | In-Place Overwriter |
| DW/DB/레이크하우스 환경
(ex. 동일 테이블, 사용자별 다른 row를 보여줘야함) | Fine-Grained Accessor for Tables |
| 버킷, 스트림, NoSQL, 클라우드 서비스 전반
(ex. job이 특정 버킷만 읽게 하고 싶을때) | Fine-Grained Accessor for Resources |
| 원본 데이터 자체를 보호해야 하는 모든 환경
(ex. 데이터가 탈취되어도 읽기 불가능하도록) | Encryptor |
| 제3자 공유, 강한 비식별 요구
(ex. 식별 가능성을 완전 차단) | Anonymizer |
| 분석 활용성과 보호를 타협해야 하는 공유 시나리오
(ex. 외부/내부 분석에 어느정도 의미를 남겨야함) | Pseudo-Anonymizer |
| 외부 API, DB 인증정보를 써야 하는 환경
(ex. 외부 API 비밀번호를 코드 내부에 X) | Secrets Pointer |
| 클라우드 매니지드 서비스, 서비스 간 통신
(ex. 클라우드 내부 서비스 접근을 패스워드 없이) | Secretless Connector |