---
layout: default
title: Airflow Task vs Dynamic DAG
---

# Airflow Task와 Dynamic DAG 어떻게 설계하는게 좋을까?

A -> B -> C 라는 Taskflow를 x와 y라는 다른 설정이 동일하게 타야할 때 1개의 DAG에 여러 Task로 나눠 처리해야할지 N개의 Dynmaic DAG으로 처리할지 고민이 들 때가 있다.

그럴때 나는 두가지 조건을 체크하는데,

- 같은 Task를 수행하는 의존성이 있는가?(e.g 모든 DAG이 하나의 테이블을 센싱해야하는가?)

그렇다면 이 경우는 여러 DAG이 불필요하게 DAG 갯수만큼 Task를 수행하여야 하니 1개의 DAG에 Task를 쪼개는 것이 맞다.

- 서로 다르지만 같은 패턴으로 DAG이 만들어지는가?(e.g 테이블을 센싱한 후 작업이 완료되면 다음 테이블을 집계한다)

그렇다면 이 경우 Dynmaic DAG을 고려할 수 있다.
