# B4 Cookie

쿠키의 `username` 값만 확인해서 admin인지 판단한다.

<img src="https://github.com/user-attachments/assets/23f870a5-064e-49df-82d4-012af542bdb1" alt="문제 첫 화면" width="800" />

## 코드 확인

```python
username = request.cookies.get('username', None)
if username:
    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not admin"}')
```

`username`이 `admin`이면 플래그를 보여준다. 쿠키 값을 바꿔보자.

## 풀이

**1.** 코드에 guest 계정이 있다.

```python
users = {
    'guest': 'guest',
    'admin': FLAG
}
```

아이디와 비밀번호가 둘 다 `guest`다. `guest / guest`로 로그인하고 개발자 도구를 켠다.

<img src="https://github.com/user-attachments/assets/2b6df182-ebf8-4abe-8e9d-58027fe37d1d" alt="guest 로그인 화면" width="680" />

<img src="https://github.com/user-attachments/assets/48a15300-ffcb-485c-87fe-19475c6721c4" alt="guest로 로그인된 화면" width="800" />

**2.** 쿠키의 `username`을 `admin`으로 바꾸고 새로고침한다.

<img src="https://github.com/user-attachments/assets/fc4da2b7-cd54-4b5f-ad61-a1277bb7a6b6" alt="username 쿠키를 admin으로 변경" width="590" />

**3.** 플래그를 확인한다.

<img src="https://github.com/user-attachments/assets/fd690dd7-7c0b-4d19-8627-ab74292eb946" alt="플래그 확인 화면" width="800" />

> 쿠키 값을 그대로 믿어서 생긴 문제다.
