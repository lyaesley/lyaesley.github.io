---
title: "setTimeout(0)의 진실: 자바스크립트 비동기 처리의 숨겨진 마법"
category: [javascript]
tag: [setTimeout]
toc: true
toc_sticky: true
published: true
---

안녕하세요, 이번 포스팅에서는 프론트엔드 개발에서 종종 만나게 되는 흥미로운 패턴인 `setTimeout(fn, 0)`에 대해 깊이 들여다보고자 합니다. 개발 경력 10년 동안 수많은 코드베이스를 다루면서, 이 패턴이 단순한 "트릭"이 아니라 자바스크립트 런타임의 깊은 이해를 바탕으로 한 강력한 도구임을 깨달았습니다.

## 들어가며: 이상한 타이머, setTimeout(0)

며칠 전, 팀의 주니어 개발자가 제게 물었습니다:

> "팀장님, 코드에서 setTimeout에 0ms를 지정해서 함수를 호출하는 걸 봤는데, 이게 의미가 있는 거예요?"

이 질문은 자바스크립트를 다루는 많은 개발자들이 한 번쯤 품게 되는 의문입니다. 타이머에 0을 지정한다면 즉시 실행되지 않을까요? 그렇다면 왜 그냥 함수를 직접 호출하지 않고 setTimeout을 사용할까요?

이 질문의 답을 찾기 위해서는 자바스크립트의 동작 원리, 특히 이벤트 루프와 실행 컨텍스트에 대한 이해가 필요합니다.

## 자바스크립트 엔진의 작동 원리

자바스크립트는 싱글 스레드 언어입니다. 이 말은 한 번에 하나의 작업만 처리할 수 있다는 의미죠. 하지만 웹 애플리케이션은 다양한 비동기 작업(네트워크 요청, 타이머, 이벤트 처리 등)을 동시에 처리해야 합니다. 이 모순을 해결하기 위해 자바스크립트 런타임은 다음과 같은 구성 요소를 가지고 있습니다:

1. **콜 스택(Call Stack)**: 현재 실행 중인 코드의 위치를 추적합니다.
2. **이벤트 루프(Event Loop)**: 콜 스택이 비어있을 때 태스크 큐에서 콜백을 가져와 실행합니다.
3. **태스크 큐(Task Queue)**:
    - **매크로태스크 큐(Macrotask Queue)**: setTimeout, setInterval, I/O 등의 콜백이 저장됩니다.
    - **마이크로태스크 큐(Microtask Queue)**: Promise, queueMicrotask 등의 콜백이 저장됩니다.
4. **웹 API**: 브라우저에서 제공하는 API (DOM, XMLHttpRequest, setTimeout 등)

이 구성 요소들이 어떻게 상호작용하는지 이해하는 것이 setTimeout(0)의 동작을 이해하는 핵심입니다.

## setTimeout(0)의 실제 동작 방식

`setTimeout(fn, 0)`이 호출되면 다음과 같은 일이 발생합니다:

1. **콜 스택에서 처리**: setTimeout 함수 자체는 동기적으로 실행됩니다.
2. **웹 API로 타이머 등록**: 브라우저는 타이머를 설정하고 콜백 함수를 웹 API 환경으로 전달합니다.
3. **콜 스택 비움**: setTimeout 함수는 콜 스택에서 제거되고, 다음 코드가 실행됩니다.
4. **매크로태스크 큐에 추가**: 타이머가 만료되면(0ms이지만 실제로는 브라우저마다 최소 지연 시간이 있음), 콜백 함수는 매크로태스크 큐에 추가됩니다.
5. **이벤트 루프에 의한 처리**:
    - 현재 실행 중인 모든 동기 코드가 완료되어 콜 스택이 비어있고,
    - 마이크로태스크 큐의 모든 태스크가 처리된 후에야,
    - 콜백 함수가 콜 스택으로 이동하여 실행됩니다.

결국, `setTimeout(fn, 0)`은 함수의 실행을 현재 실행 컨텍스트의 끝으로 지연시키는 효과가 있습니다. 다른 모든 동기 코드와 마이크로태스크가 완료된 후, 다음 "틱(tick)"에서 실행되도록 예약하는 것이죠.

## 왜 setTimeout(0)을 사용하는가?

수년간의 개발 경험에서, 나는 setTimeout(0)이 다음과 같은 상황에서 매우 유용하다는 것을 발견했습니다:

1. **DOM 업데이트 보장**:
```javascript
// DOM 요소 추가
   container.innerHTML = complexHTML;
   
   // DOM이 렌더링된 후 실행되길 원하는 코드
   setTimeout(() => {
     initEventHandlers();
     adjustLayout();
   }, 0);
```


2. **실행 순서 제어**:
```javascript
function processLargeData(data) {
     // 첫 부분 처리
     const firstPart = processFirstPart(data);
     
     // UI 블로킹 방지를 위해 다음 처리를 다음 틱으로 연기
     setTimeout(() => {
       const secondPart = processSecondPart(data);
       // ...
     }, 0);
     
     return firstPart;
   }
```


3. **이벤트 루프 활용**:
```javascript
function deferExecution() {
     const result = heavyComputation();
     
     // 현재 실행 컨텍스트 완료 후 콜백 실행
     setTimeout(() => {
       notifyComplete(result);
     }, 0);
     
     return "Processing...";
   }
```


4. **디바운싱과 스로틀링**:
   이벤트 핸들러에서 성능 최적화를 위해 사용됩니다.

## setTimeout(0)의 깊은 이해: 실제 테스트 코드

이론적인 설명보다 실제 코드를 통한 이해가 더 명확할 때가 많습니다. 다음은 제가 자바스크립트의 비동기 동작을 이해하기 위해 작성한 테스트 코드입니다:

```javascript
/**
 * setTimeout(fn, 0) 실행 순서 테스트
 * 이 코드는 자바스크립트의 이벤트 루프, 콜 스택, 태스크 큐의 동작을 보여줍니다.
 */

// 로그 출력 함수 - 타임스탬프 추가
function logWithTime(message, type = '') {
  const now = new Date();
  const time = `${now.getMinutes()}:${now.getSeconds()}.${now.getMilliseconds().toString().padStart(3, '0')}`;
  
  switch(type) {
    case 'sync':
      console.log(`[${time}] %c${message}`, 'color: blue');
      break;
    case 'macro':
      console.log(`[${time}] %c${message}`, 'color: red');
      break;
    case 'micro':
      console.log(`[${time}] %c${message}`, 'color: purple');
      break;
    default:
      console.log(`[${time}] ${message}`);
  }
}

// 테스트 함수
function runSetTimeoutTest() {
  console.log('===== setTimeout(0) 실행 순서 테스트 시작 =====');
  
  // 1. 동기 코드 시작
  logWithTime('1. 동기 코드 실행 시작', 'sync');
  
  // 2. setTimeout 등록 (매크로태스크)
  logWithTime('2. setTimeout(0) 호출 - 매크로태스크 큐에 추가됨', 'sync');
  setTimeout(() => {
    logWithTime('6. setTimeout(0) 콜백 실행 - 콜 스택이 비워지고 마이크로태스크가 모두 처리된 후 실행됨', 'macro');
    
    logWithTime('7. setTimeout(0) 내부에서 새로운 Promise 생성', 'macro');
    Promise.resolve().then(() => {
      logWithTime('8. setTimeout 내부 Promise의 콜백 실행 - 새로운 마이크로태스크', 'micro');
    });
    
    logWithTime('9. setTimeout(0) 콜백 종료', 'macro');
  }, 0);
  
  // 3. Promise 등록 (마이크로태스크)
  logWithTime('3. Promise 생성 - 마이크로태스크 큐에 추가됨', 'sync');
  Promise.resolve().then(() => {
    logWithTime('5. Promise 콜백 실행 - 현재 이벤트 루프의 마이크로태스크 단계에서 실행됨', 'micro');
  });
  
  // 4. 두 번째 동기 작업
  logWithTime('4. 동기 코드 실행 종료 - 콜 스택에서 모든 동기 코드가 완료됨', 'sync');
  
  // 추가적인 setTimeout으로 실행 순서 확인
  setTimeout(() => {
    logWithTime('10. 두 번째 setTimeout(10) 콜백 실행', 'macro');
    
    setTimeout(() => {
      logWithTime('12. 세 번째 setTimeout(0) 콜백 실행', 'macro');
    }, 0);
    
    Promise.resolve().then(() => {
      logWithTime('11. 두 번째 setTimeout 내부의 Promise 콜백 실행', 'micro');
    });
    
  }, 10);
  
  console.log('===== 테스트 코드 등록 완료, 이벤트 루프에 의해 처리되는 순서 확인 =====');
}

// HTML 환경에서 함수 실행
function createHTMLElement() {
  // HTML 요소를 동적으로 생성하는 함수
  const div = document.createElement('div');
  div.id = 'testDiv';
  div.innerHTML = '테스트용 DIV 요소';
  document.body.appendChild(div);
  logWithTime('HTML 요소 생성 완료', 'sync');
  
  // setTimeout 내에서 DOM 조작
  setTimeout(() => {
    logWithTime('setTimeout 내에서 DOM 요소 조작', 'macro');
    div.innerHTML = '변경된 내용';
    
    // 이벤트 리스너 등록
    div.addEventListener('click', () => {
      logWithTime('클릭 이벤트 발생', 'sync');
    });
    
    // 테스트를 위해 프로그래밍 방식으로 클릭 이벤트 발생
    logWithTime('프로그래밍 방식으로 클릭 이벤트 발생시킴', 'macro');
    div.click();
  }, 0);
  
  logWithTime('DOM 조작 코드 등록 완료', 'sync');
}

// 메모리 영향 테스트
function testMemoryUsage() {
  logWithTime('메모리 사용량 테스트 시작', 'sync');
  
  // 많은 수의 setTimeout 등록
  for(let i = 0; i < 1000; i++) {
    setTimeout(() => {
      // 아무것도 하지 않는 콜백
    }, 0);
  }
  
  logWithTime('1000개의 setTimeout(0) 등록 완료', 'sync');
  
  // 많은 수의 Promise 등록
  for(let i = 0; i < 1000; i++) {
    Promise.resolve().then(() => {
      // 아무것도 하지 않는 콜백
    });
  }
  
  logWithTime('1000개의 Promise 등록 완료', 'sync');
  
  setTimeout(() => {
    logWithTime('모든 콜백이 처리된 후 실행됨', 'macro');
  }, 100);
}

// 세 가지 테스트 실행
function runAllTests() {
  runSetTimeoutTest();
  
  setTimeout(() => {
    console.log('\n===== DOM 조작 테스트 =====');
    if (typeof document !== 'undefined') {
      createHTMLElement();
    } else {
      console.log('브라우저 환경이 아니므로 DOM 테스트를 건너뜁니다.');
    }
  }, 500);
  
  setTimeout(() => {
    console.log('\n===== 메모리 사용량 테스트 =====');
    testMemoryUsage();
  }, 1000);
  
  setTimeout(() => {
    console.log('\n===== 테스트 결과 요약 =====');
    console.log('1. 실행 순서: 동기 코드 → 마이크로태스크(Promise) → 매크로태스크(setTimeout)');
    console.log('2. setTimeout(0)은 실제로 0ms 후에 실행되지 않고, 콜 스택이 비워진 후 다음 이벤트 루프에서 실행됨');
    console.log('3. 마이크로태스크는 현재 이벤트 루프 틱에서 처리됨');
    console.log('4. 각 매크로태스크 이후에는 렌더링이 일어날 수 있음');
    console.log('===== 테스트 종료 =====');
  }, 1500);
}

// 테스트 실행
runAllTests();
```


이 코드를 브라우저 콘솔에 직접 붙여넣고 실행해 보시면, 자바스크립트의 비동기 실행 순서를 명확하게 이해할 수 있습니다.

## 코드 실행 결과와 해석

이 테스트 코드를 실행하면 다음과 같은 순서로 로그가 출력됩니다:

1. 먼저 모든 동기 코드가 실행됩니다 (파란색 로그).
2. 다음으로 마이크로태스크 큐의 작업(Promise)이 처리됩니다 (보라색 로그).
3. 마지막으로 매크로태스크 큐의 작업(setTimeout)이 실행됩니다 (빨간색 로그).

특히 주목할 점은 다음과 같습니다:

- Promise 콜백은 동기 코드 직후에 실행됩니다.
- setTimeout(0) 콜백은 모든 동기 코드와 마이크로태스크가 완료된 후에야 실행됩니다.
- setTimeout 내부에서 생성된 Promise는 해당 setTimeout 콜백이 완료된 직후에 실행됩니다.

이러한 실행 순서를 이해하면, setTimeout(0)을 효과적으로 활용할 수 있습니다.

## 실무에서의 setTimeout(0) 활용 사례

개발자로서 10년간 일하면서, 저는 다양한 실무 상황에서 setTimeout(0)을 활용했습니다:

1. **SPA에서의 라우팅 후 DOM 조작**:
```javascript
// 라우트 변경 후 DOM이 업데이트될 시간을 주고 스크롤 위치 조정
   router.afterEach((to, from) => {
     setTimeout(() => {
       window.scrollTo(0, 0);
     }, 0);
   });
```


2. **대용량 데이터 처리와 UI 응답성 유지**:
```javascript
function processLargeDataset(items) {
     const results = [];
     const chunks = splitIntoChunks(items, 100);
     
     function processChunk(index) {
       if (index >= chunks.length) {
         finishProcessing(results);
         return;
       }
       
       // 현재 청크 처리
       const partialResults = processItems(chunks[index]);
       results.push(...partialResults);
       
       // 다음 청크를 다음 이벤트 루프에서 처리
       setTimeout(() => {
         processChunk(index + 1);
       }, 0);
     }
     
     processChunk(0);
   }
```


3. **이벤트 핸들러의 안전한 등록**:
```javascript
function renderComplexTemplate(container, data) {
     // HTML 생성 및 삽입
     container.innerHTML = generateHTML(data);
     
     // DOM이 업데이트된 후 이벤트 핸들러 등록
     setTimeout(() => {
       container.querySelectorAll('.interactive-element').forEach(el => {
         el.addEventListener('click', handleInteraction);
       });
     }, 0);
   }
```


## 더 현대적인 대안: queueMicrotask와 requestAnimationFrame

최신 자바스크립트에서는 setTimeout(0) 대신 사용할 수 있는 더 명확한 API들이 제공됩니다:

1. **queueMicrotask**: 현재 이벤트 루프 틱의 끝에서, 다음 매크로태스크 전에 실행됩니다.
```javascript
queueMicrotask(() => {
     // 동기 코드와 현재 마이크로태스크 이후, 다음 매크로태스크 전에 실행
   });
```


2. **requestAnimationFrame**: 다음 리페인트 전에 실행됩니다.
```javascript
requestAnimationFrame(() => {
     // 다음 화면 업데이트 전에 실행
   });
```


3. **Promise.resolve().then()**: queueMicrotask와 유사하게 동작합니다.
```javascript
Promise.resolve().then(() => {
     // 마이크로태스크 큐에 추가됨
   });
```


각 API는 서로 다른 실행 시점을 가지므로, 요구사항에 맞는 것을 선택하는 것이 중요합니다.

## 결론: setTimeout(0)의 가치

setTimeout(0)는 단순한 "해킹"이 아니라, 자바스크립트 런타임의 작동 방식에 기반한 유용한 기법입니다. 이를 올바르게 활용하면 다음과 같은 이점을 얻을 수 있습니다:

1. DOM 업데이트와 이벤트 핸들러 등록의 적절한 순서 보장
2. 큰 작업을 작은 단위로 분할하여 UI 응답성 유지
3. 이벤트 루프의 동작을 활용한 실행 흐름 제어

물론, 최신 API들을 활용하면 의도를 더 명확히 표현할 수 있지만, setTimeout(0)의 기본 원리를 이해하는 것은 여전히 중요합니다.

10년 동안 프론트엔드 개발자로 일하면서, 나는 한 가지 진리를 깨달았습니다: 자바스크립트의 동작 원리를 깊이 이해할수록, 더 효과적이고 강력한 코드를 작성할 수 있다는 것입니다.

이 포스팅이 setTimeout(0)의 신비를 풀고, 여러분의 코드에 새로운 가능성을 열어주었기를 바랍니다. 여러분의 경험과 의견도 댓글로 공유해 주세요!

---

**참고 자료:**
1. [MDN Web Docs: setTimeout()](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout)
2. [Event Loop and the Big Picture](https://javascript.info/event-loop)
3. [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
4. [HTML Living Standard: Timers](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers)