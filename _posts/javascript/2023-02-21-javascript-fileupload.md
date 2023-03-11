---
title: "[javascript] 파일업로드 SIZE/가로세로 체크"
category: [javascript]
tag: [fileupload, image]
toc: true
toc_sticky: true
published: true
---

### 파일 용량 체크

```javascript
// HTML에서 파일 업로드 input을 가져옵니다.
var fileInput = document.getElementById('file-input');

// 파일 업로드 input의 change 이벤트를 감지합니다.
fileInput.addEventListener('change', function() {
  // 파일 객체를 가져옵니다.
  var file = fileInput.files[0];

  // 파일의 크기를 체크합니다.
  if (file.size > 500 * 1024) {
    // 파일 크기가 500KB를 넘는 경우 처리합니다.
    alert('이미지 파일의 크기는 최대 500KB 이하이어야 합니다.');
    fileInput.value = '';
  }
});
```

### 가로/세로 사이즈 체크

`<input type="file">`의 change 이벤트리스너를 등록해서 구현  

아래 코드를 활용해서 이미지업로드 완료후 체크하는 방식도 구현 가능

```javascript
    // HTML에서 파일 업로드 input을 가져옵니다.
    var fileInput = document.getElementById('file-input');
    
    // 파일 업로드 input의 change 이벤트를 감지합니다.
    fileInput.addEventListener('change', function() {
      // 파일을 읽기 위한 FileReader 객체를 생성합니다.
      var reader = new FileReader();
    
      // 파일 로드가 완료되면 실행될 콜백 함수를 정의합니다.
      reader.onload = function(event) {
        // 이미지 객체를 생성합니다.
        var image = new Image();
    
        // 이미지 객체의 로드가 완료되면 실행될 콜백 함수를 정의합니다.
        image.onload = function() {
          // 이미지 객체의 너비와 높이를 가져옵니다.
          var width = this.width;
          var height = this.height;
    
          // 이미지의 크기를 체크합니다.
          if (width > 1024 || height > 1024) {
            // 이미지가 1024px 보다 크면 에러 처리합니다.
            alert('이미지의 크기는 최대 1024px x 1024px 이하이어야 합니다.');
            fileInput.value = '';
          }
        };
    
        // 이미지 객체에 파일 데이터를 설정하고 로드합니다.
        image.src = event.target.result;
      };
    
      // FileReader 객체에 파일 데이터를 설정하고 로드합니다.
      reader.readAsDataURL(fileInput.files[0]);
    });
```
