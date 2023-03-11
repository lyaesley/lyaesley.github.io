---
title: "[Codility] Lesson7(Stacks and Queues) StoneWall"
category: [Algorithm]
tag: [codility]
toc: true
toc_sticky: true
published: true
---

# [StoneWall](https://app.codility.com/programmers/lessons/7-stacks_and_queues/stone_wall/)

### 문제

- Cover "Manhattan skyline" using the minimum number of rectangles.
- 효율적인 알고리즘 작성

### 문제 풀이

- Stack 사용
- 블록의 높이를 저장하기 위해 스택을 초기화한다.
- 배열 H의 각 높이 h에 대해 다음을 수행한다. 
    - 스택의 상단이 h보다 높으면 상단 요소를 pop하고 블록 카운트를 추가한다.
    - 스택의 상단이 h보다 낮으면 스택에 h를 push 한다.
- 스택이 비어있지않으면 개수만큼 블록을 더한다.


### CODE

<script src="https://gist.github.com/lyaesley/ff8a2620ac0c45e367c6c2bb63e5971e.js"></script>

### 결과

- 시간복잡도 O(N)
- Task Score 100%
- Correctness 100%
- Performance 100%
- [https://app.codility.com/demo/results/trainingU6AA2K-DNF/](https://app.codility.com/demo/results/trainingU6AA2K-DNF/)

