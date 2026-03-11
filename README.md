# growingstudy (성장형 스터디 서비스)

스터디 그룹 관리 및 학습 진행 추적을 위한 백엔드 애플리케이션입니다.

## 팀원 소개 (가나다 순)

- 김지용: 회원 기능, 시큐리티 구성 및 배포 서버 인프라, CI / CD 파이프라인
- 이단규: 스터디 그룹 기능, 세션 기능, 그룹 공지사항 기능, 학습 시간 기능, ERD 설계
- 정다경: 체크 리스트 기능, 과제물 기능

## 기술 스택

| 분류            | 기술                       |
|---------------|--------------------------|
| Language      | Java 21                  |
| Framework     | Spring Boot 3.5.6        |
| Security      | Spring Security, JWT     |
| Persistence   | Spring Data JPA          |
| Database      | MySQL (H2 for dev/test)  |
| Cache         | Redis (Refresh Token 저장) |
| Build Tool    | Gradle                   |
| CI / CD       | Github Actions           |
| Development   | IntelliJ IDEA            |
| Documentation | Notion (API Spec, ERD)   |
| Collaboration | Discord, Notion          |

## 배포 서버 인프라 (AWS)

<img width="7077" height="5691" alt="image" src="https://github.com/user-attachments/assets/a16e4627-554a-439a-aaff-8b6c68f7a976" />

| 분류             | 기술                          |
|----------------|-----------------------------|
| Compute        | EC2 (Ubuntu 24.04 LTS)      |
| Database       | RDS (MySQL 8.0.43)          |
| Cache          | ElastiCache (Redis 7.1.0)   |
| Storage        | S3 (파일 업로드, 배포 아티팩트 관리)     |
| Access Control | IAM (관리자 / 개발자 / EC2 권한 분리) |
| Secure Access  | Systems Manager (SSM 포트포워딩) |

## 기능 개요

### 회원 (Authentication)

| Method | Endpoint              | 설명                                 |
|--------|-----------------------|------------------------------------|
| POST   | `/api/auth/login`     | 로그인 (JSON 기반 인증, JWT 토큰 발급)        |
| POST   | `/api/auth/refresh`   | 토큰 재발급 (Refresh Token으로 새 토큰 쌍 발급) |
| POST   | `/api/auth/logout`    | 로그아웃 (Refresh Token 무효화)           |
| POST   | `/api/auth/register`  | 회원가입                               |
| GET    | `/api/auth/me`        | 내 정보 조회                            |
| GET    | `/api/auth/mycoffees` | 내 커피 도감 조회                         |

### 스터디 그룹 (Study Group)

| Method | Endpoint                          | 설명           |
|--------|-----------------------------------|--------------|
| GET    | `/api/studyrooms`                 | 그룹 목록 조회     |
| POST   | `/api/studyrooms`                 | 그룹 생성        |
| GET    | `/api/studyrooms/{groupId}`       | 그룹 상세 정보 조회  |
| POST   | `/api/studyrooms/join`            | 초대 코드로 그룹 참가 |
| DELETE | `/api/studyrooms/{groupId}/leave` | 그룹 나가기       |

### 세션 (Session)

| Method | Endpoint                                         | 설명          |
|--------|--------------------------------------------------|-------------|
| POST   | `/api/studyrooms/{groupId}/sessions`             | 세션 생성       |
| GET    | `/api/studyrooms/{groupId}/sessions/{sessionId}` | 세션 정보 조회    |
| GET    | `/api/studyrooms/{groupId}/sessions/progress`    | 세션 진행 상황 조회 |

### 체크리스트 (Checklist)

| Method | Endpoint                                                    | 설명          |
|--------|-------------------------------------------------------------|-------------|
| GET    | `/api/studyrooms/{groupId}/sessions/{sessionId}/checklists` | 체크리스트 목록 조회 |
| POST   | `/api/studyrooms/{groupId}/sessions/{sessionId}/checklists` | 체크리스트 생성    |

### 과제물 (Submission)

| Method | Endpoint                                                                                             | 설명                 |
|--------|------------------------------------------------------------------------------------------------------|--------------------|
| GET    | `/api/studyrooms/{groupId}/sessions/{sessionId}/checklists/{checklistId}`                            | 과제물 조회             |
| POST   | `/api/studyrooms/{groupId}/sessions/{sessionId}/checklists/{checklistId}`                            | 과제물 제출 (파일 업로드 지원) |
| PATCH  | `/api/studyrooms/{groupId}/sessions/{sessionId}/checklists/{checklistId}/submissions/{submissionId}` | 과제물 인증 처리          |

### 그룹 공지사항 (Notice)

| Method | Endpoint                           | 설명              |
|--------|------------------------------------|-----------------|
| GET    | `/api/studyrooms/{groupId}/notice` | 현재 공지(가장 최신) 조회 |
| POST   | `/api/studyrooms/{groupId}/notice` | 공지 생성           |

### 학습 시간 (Study Time)

| Method | Endpoint                                          | 설명          |
|--------|---------------------------------------------------|-------------|
| POST   | `/api/studyrooms/{groupId}/{sessionId}/study`     | 학습 시간 기록    |
| GET    | `/api/studyrooms/{groupId}/{sessionId}/studylogs` | 학습 시간 로그 조회 |

## 기술적 특징

### JWT 기반 인증 아키텍처

커스텀 Security Filter 체인을 통한 JWT 인증 흐름을 구현하였습니다.

1. **JsonAuthenticationProcessingFilter**: `/api/auth/login` 요청을 처리하며, JSON 형식의 이메일/비밀번호를 받아 인증 후 Access Token과 Refresh Token을 발급합니다.

2. **RegenerateTokensFilter**: `/api/auth/refresh` 요청을 처리하며, Refresh Token 검증 후 새로운 토큰 쌍을 발급합니다.

3. **토큰 관리**:
   - Access Token: 60분 유효
   - Refresh Token: Redis에 저장, 사용자당 하나만 유효 (새 토큰 발급 시 기존 토큰 무효화)
   - RSA 2048-bit 키 쌍을 사용한 JWT 서명

### 도메인 중심 패키지 구조

```
com.example.growingstudy
├── auth        # 회원가입, 인증, 계정 관리
├── security    # JWT 인프라, 필터, 보안 설정
├── studygroup  # 스터디 그룹 생성 및 관리
├── session     # 스터디 세션, 체크리스트, 과제물
├── groupsub    # 공지사항, 학습 시간 기록
├── coffee      # 커피 기반 리워드 시스템
└── global      # 전역 예외 처리, 전역 서비스 (예: S3)
```

### 엔티티 관계

- `Account` ↔ `GroupMember` ↔ `StudyGroup`: 다대다 관계 (조인 엔티티 사용)
- `StudyGroup` → `Session` → `Checklist`: 계층적 구조
- `Account` + `Checklist` → `Submission`: 멤버별 체크리스트 제출
- `StudyGroup` → `TotalStudyTime`, `StudyTimeLog`: 학습 시간 집계 및 로그
- `StudyGroup` → `GroupCoffee`: 그룹별 커피 리워드 진행 상황
