# session-basic
### 1. VM에 접속한다

![image.png](%E1%84%83%E1%85%B3%E1%84%85%E1%85%B5%E1%86%B7%E1%84%92%E1%85%A2%E1%86%A8/image.png)

### 2. app.py를 분석

```python
# (생략)

session_storage = {}

@app.route('/')
def index():
    session_id = request.cookies.get('sessionid', None)

    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html')

    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not admin"}')

# (생략)

@app.route('/admin')
def admin():
    #session_id = request.cookies.get('sessionid', None)
    #username = session_storage[session_id]
    #if username != 'admin':
    #    return render_template('index.html')

    return session_storage

if __name__ == '__main__':
    import os

    session_storage[os.urandom(32).hex()] = 'admin'
    print(session_storage)

    # (생략)
```

소스 코드를 분석한 결과, 

/admin 경로의 관리자 권한 검증 코드가 주석 처리되어 있고,

returnsession_storage를 통해 전체 세션 정보가 반환되는 것을 확인하였다. 

**`/admin`에 접속하여 시도해보자.**

### 3. /admin 주소에 접속해서 Session id 정보 받기

![image.png](%E1%84%83%E1%85%B3%E1%84%85%E1%85%B5%E1%86%B7%E1%84%92%E1%85%A2%E1%86%A8/image%201.png)

### 4. 개발자 도구에 해당 관리자 session id를 입력한다.

![image.png](%E1%84%83%E1%85%B3%E1%84%85%E1%85%B5%E1%86%B7%E1%84%92%E1%85%A2%E1%86%A8/image%202.png)

### 5. 새로고침해서 flag를 확인한다.

![image.png](%E1%84%83%E1%85%B3%E1%84%85%E1%85%B5%E1%86%B7%E1%84%92%E1%85%A2%E1%86%A8/image%203.png)
