---
title: "이커머스 주문 시스템 DB 모델링 및 쿼리 튜닝"
excerpt: "대용량 주문 처리를 위한 MySQL 설계 및 Slow Query 개선 사례"
categories:
  - Project
tags:
  - MySQL
  - Tuning
  - Modeling
last_modified_at: 2024-12-02
---

## 1. 프로젝트 개요
* **주제:** 온라인 쇼핑몰 주문/결제 시스템 DB 구축
* **기간:** 2024.11 ~ 2024.12
* **역할:** Database Engineer (기여도 100%)
* **사용 기술:** MySQL 8.0, AWS RDS, Python (Data Generator)

## 2. ERD 설계 (데이터 모델링)
> 비즈니스 로직을 반영하여 제3정규형을 준수하되, 조회 성능을 위해 주문 이력 테이블은 반정규화를 수행했습니다.

*(여기에 나중에 ERD 이미지 파일을 넣을 겁니다)*

## 3. 핵심 문제 해결 (Performance Tuning)

### 3.1 문제 상황
주문 내역 조회 페이지에서 로딩 시간이 **3.5초** 이상 소요되는 현상 발생. (데이터 100만 건 기준)

### 3.2 원인 분석
`Explain` 실행 결과, `order_date` 컬럼에 인덱스가 없어 **Full Table Scan**이 발생하고 있었습니다.

```sql
-- 튜닝 전 쿼리 (Full Scan)
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';
