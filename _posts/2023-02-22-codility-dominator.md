---
title: "[Codility] Lesson8(Leader) Dominator"
category: [Algorithm]
tag: [codility]
toc: true
toc_sticky: true
published: true
---

# [Dominator](https://app.codility.com/programmers/lessons/8-leader/dominator/)

### 문제

- Find an index of an array such that its value occurs at more than half of indices in the array.
- 효율적인 알고리즘 작성

### 문제 풀이

- Stack 사용
- 스택이 비어 있으면 요소를 스택으로 푸시합니다.
- 스택 맨 위에 있는 요소가 현재 요소와 같으면 현재 요소를 스택으로 푸시합니다.
- 그렇지 않으면 스택에서 맨 위 요소를 팝합니다.
첫 번째 반복 후 스택이 비어 있지 않으면 맨 위 요소가 후보 도미네이터입니다. 그런 다음 배열을 두 번째로 반복하여 후보 도미네이터의 발생 횟수를 계산합니다. 후보 도미네이터가 배열 요소의 절반 이상에서 발생하는 경우 후보 도미네이터 발생의 인덱스를 반환합니다. 그렇지 않으면 -1을 반환합니다.


### CODE

```java
import java.util.*;

class Solution {
    public int solution(int[] A) {
        Stack<Integer> stack = new Stack<>();
        for (int i = 0; i < A.length; i++) {
            if (stack.isEmpty()) {
                stack.push(A[i]);
            } else if (stack.peek() == A[i]) {
                stack.push(A[i]);
            } else {
                stack.pop();
            }
        }
        if (stack.isEmpty()) {
            return -1;
        }
        int candidate = stack.peek();
        int count = 0;
        for (int i = 0; i < A.length; i++) {
            if (A[i] == candidate) {
                count++;
                if (count > A.length / 2) {
                    return i;
                }
            }
        }
        return -1;
    }
}
```


### 결과

- 시간복잡도 O(N*log(N)) or O(N)
- Task Score 100%
- Correctness 100%
- Performance 100%
- [https://app.codility.com/demo/results/trainingD7S4E5-GX5/](https://app.codility.com/demo/results/trainingD7S4E5-GX5/)

