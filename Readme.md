# Jekyll 프로젝트 실행 가이드

이 문서는 Jekyll을 설치하고 실행하는 방법에 대해 설명합니다.

---

## 1. Jekyll 설치 확인

Jekyll이 설치되어 있는지 확인하려면 다음 명령어를 실행합니다:

```bash
jekyll -v
```

Jekyll 버전이 출력되지 않으면 설치가 필요합니다.

### Jekyll 설치 방법

1. **Ruby와 Bundler 설치**
    - Jekyll은 Ruby 기반으로 동작하므로 Ruby와 Bundler가 필요합니다.
    - Ruby가 설치되지 않은 경우 [Ruby 설치 가이드](https://www.ruby-lang.org/)를 참고하세요.

2. **Jekyll 및 Bundler 설치**
   아래 명령어를 실행하여 설치합니다:

   ```bash
   gem install jekyll bundler
   ```

---

## 2. Jekyll 프로젝트 생성

새로운 Jekyll 프로젝트를 생성하려면 다음 명령어를 실행하세요:

```bash
jekyll new my-blog
cd my-blog
```

이 명령은 `my-blog` 디렉터리에 기본 Jekyll 템플릿을 생성합니다.

---

## 3. Jekyll 프로젝트 실행

Jekyll 프로젝트를 실행하려면 프로젝트 디렉터리(`my-blog`)에서 다음 명령어를 사용하세요:

### Bundler를 사용하는 경우

```bash
bundle exec jekyll serve
```

### Bundler를 사용하지 않는 경우

```bash
jekyll serve
```

### 실행 확인
- 기본적으로 [http://localhost:4000](http://localhost:4000)에서 사이트를 확인할 수 있습니다.

---

## 4. 추가 실행 옵션

실행 시 필요에 따라 다음 옵션을 사용할 수 있습니다:

| 옵션                 | 설명                                             |
|----------------------|------------------------------------------------|
| `--drafts`           | 초안 게시물도 포함하여 빌드합니다.                  |
| `--host`             | 특정 호스트 주소로 서버를 실행합니다.               |
| `--port`             | 실행 포트를 지정합니다. 기본값은 `4000`입니다.        |
| `--no-watch`         | 파일 변경 감지를 비활성화합니다.                    |

### 예시

```bash
jekyll serve --drafts --host 127.0.0.1 --port 5000
```

이 명령어는 초안을 포함하여 빌드하며, `http://127.0.0.1:5000`에서 확인할 수 있습니다.

---

## 5. 참고 자료

- [Jekyll 공식 문서](https://jekyllrb.com/)
- [Ruby 설치 가이드](https://www.ruby-lang.org/)

---
