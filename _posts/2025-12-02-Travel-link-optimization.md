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
* **기간:** 2025.09 ~ 2025.12
* **기술 스택:** Python FastAPI, SQLAlchemy, **MySQL 8.0**, AWS RDS
* **나의 역할:** Database Architect & Backend Developer

## 2. 핵심 아키텍처 및 DB 설계 (The Architect)

### 2.1 AI 비정형 데이터를 위한 Hybrid 설계 (RDBMS + JSON)
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
