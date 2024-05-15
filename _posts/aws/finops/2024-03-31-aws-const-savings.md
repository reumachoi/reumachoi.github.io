---
title: OpsNow360 분석을 통한 비용절감
author: rumi
date: 2024-03-31 11:33:00 +0900
categories: [AWS, Cost Savings]
tags: [opsnow360]
---

# OpsNow360 분석을 통한 비용절감

## 많이 나오는 리소스 확인
1. 상세 데이터 엑셀파일 추출
2. 피봇테이블 추출해서 USETYPE으로 많이나오는 Type 확인
3. 2번에서 알아낸 타입만 필터링해서 피봇테이블 만들기
    > 행: startDate | 열: ResourceName | 값: totalCoast