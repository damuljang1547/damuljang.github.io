---
title: "Travel Link - MySQL JSON 타입을 활용한 스키마 설계 및 동시성 제어"
excerpt: "AI 생성 데이터의 효율적 저장(JSON)과 선착순 모집 Race Condition 해결 (MySQL 8.0, SQLAlchemy)"
categories:
  - Project
tags:
  - MySQL 8.0
  - JSON Modeling
  - Concurrency Control
  - FastAPI
last_modified_at: 2025-12-02
---

## 1. 프로젝트 개요
* **서비스명:** Travel Link (AI 기반 여행 계획 공유 플랫폼)
* **기간:** 2025.03 ~ 2025.12
* **기술 스택:** Python FastAPI, SQLAlchemy, **MySQL 8.0**, AWS RDS
* **나의 역할:** Database Architect & Backend Developer

## 2. 요구사항 분석 및 데이터 모델링 (Requirements & Modeling)

### 2.1 비즈니스 요구사항과 데이터 설계
Travel Link는 신뢰 기반의 동행 모집 플랫폼이므로, **"인증된 사용자"**와 **"데이터 무결성"**을 최우선으로 고려하여 요구사항을 도출했습니다.

| 도메인 | 요구사항 (Business Logic) | DB 설계 반영 (Data Strategy) |
| :--- | :--- | :--- |
| **회원 (User)** | 모든 활동(생성, 신청)은 회원만 가능하다. | `plans`, `applications` 등 주요 테이블의 `user_id`를 **NOT NULL**로 설정하여 참조 무결성 강제. |
| **계획 (Plan)** | AI가 생성하는 일정은 길이가 가변적이다. | 정형 데이터(제목, 지역)와 비정형 데이터(AI 일정)를 분리하여 **Hybrid 설계(RDBMS + JSON)** 적용. |
| **참여 (Join)** | 참여는 [신청 → 승인] 단계를 거치며, 정원을 초과할 수 없다. | 신청(`plan_applications`)과 확정(`plan_participants`) 테이블을 분리하고, **비관적 락(Lock)**으로 동시성 제어. |

> **📌 설계 의도: 데이터 품질 확보**
> "익명 게시판이 아닌, 실제 만남이 이루어지는 서비스입니다. 따라서 **모든 테이블에 유저 식별자(FK)를 필수 조건으로 설정**하여, 데이터의 소유권을 명확히 하고 추후 발생할 수 있는 악성 유저 이슈에 대비했습니다."

### 2.2 Entity Relationship Diagram (ERD)
사용자(User)를 중심으로 여행 계획(Plan)이 생성되고, 이에 대한 참여 신청(Application)과 확정(Participant)이 이루어지는 흐름을 시각화했습니다.

![Travel Link ERD](/assets/TL-ERD.png)

---

## 3. 핵심 아키텍처 및 DB 설계 (The Architect)

### 3.1 AI 비정형 데이터를 위한 Hybrid 설계 (RDBMS + JSON)
여행 계획(`itinerary`)은 AI(Gemini)가 생성하므로, 여행 일수와 방문지 개수가 매번 달라지는 가변적인 계층 구조(Date -> Time -> Activity)를 가집니다.

* **문제 (Problem):** 이를 정규화(Normalization)하여 `Plan_Days`, `Plan_Activities` 테이블로 쪼갤 경우, 상세 조회 시 수백 개의 Row를 `JOIN`해야 하므로 I/O 부하가 큼.
* **해결 (Solution):** **MySQL 8.0의 JSON Data Type**을 도입하여 `plans` 테이블의 `itinerary` 컬럼에 구조화된 JSON을 직접 저장.
* **무결성 확보:** 단순 텍스트 저장이 아닌, 데이터 무결성을 위해 테이블 생성 시 **Check Constraint**를 적용함.

```sql
-- 실제 적용된 DDL (Dump 파일 발췌)
CREATE TABLE `plans` (
  `id` int NOT NULL AUTO_INCREMENT,
  `title` varchar(255) NOT NULL,
  `itinerary` JSON, -- AI 결과값 저장 (Native JSON Type)
  `participants` int DEFAULT 1,
  `capacity` int DEFAULT 4,
  -- JSON 유효성 검사 제약조건 추가
  CONSTRAINT `plans_chk_1` CHECK (json_valid(`itinerary`)),
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
