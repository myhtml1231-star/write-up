# B2 · Simple Note Manager

### 1. VM에 접속한다

문제 페이지에 접속하면 노트를 생성·수정·삭제하고 백업하는 기능이 있다. `Create Note`로 노트를 하나 생성한 뒤 백업 기능의 코드를 확인한다.

### 2. 코드 분석 — OS Command Injection이 발생하는 과정

**① 노트를 생성하면 `notes` 딕셔너리에 저장된다.**

```python
new_note_id = 0
notes = {}

def create_note(content):
    global new_note_id
    with lock:
        note_id = new_note_id
        new_note_id += 1
        notes[note_id] = content
        return notes[note_id]
```

`/create_note`에 제출한 `content`가 `notes`에 저장된다. 노트가 최소 1개 있어야 백업 기능을 사용할 수 있다.

**② `/backup_notes`에 GET 요청을 보내면 `backup-timestamp` 쿠키가 설정된다.**

```python
@app.route('/backup_notes', methods=['GET'])
def get_backup_notes():
    print(len(notes), flush=True)
    if len(notes) == 0:
        abort(404)
    page = render_template('backup_notes.html')
    resp = make_response(page)
    resp.set_cookie('backup-timestamp', f'{time.time()}')
    return resp
```

백업 페이지는 노트가 없으면 `404`를 반환한다. 노트가 있으면 현재 시간을 `backup-timestamp` 쿠키로 저장한다.

**③ `/backup_notes`에 POST 요청을 보내면 쿠키 값을 그대로 백업 함수에 전달한다.**

```python
@app.route('/backup_notes', methods=['POST'])
def post_backup_notes():
    if len(notes) == 0:
        abort(404)
    backup_timestamp = request.cookies.get('backup-timestamp', f'{time.time()}')
    if not isinstance(backup_timestamp, str):
        abort(400)
    backup_notes(backup_timestamp)
    return redirect(url_for('get_index'))
```

`request.cookies.get()`은 클라이언트가 보낸 쿠키 값을 읽는다. `isinstance(..., str)` 검사는 문자열인지 확인할 뿐, 쉘 특수문자를 제한하지 않는다. 따라서 사용자가 쿠키를 수정하면 그 값이 `backup_notes()`로 전달된다.

**④ `backup_notes()`에서 쿠키 값을 쉘 명령어에 직접 삽입한다.**

```python
def backup_notes(timestamp):
    with lock:
        with open('./tmp/notes.tmp', 'w') as f:
            f.write(repr(notes))
        subprocess.Popen(f'cp ./tmp/notes.tmp /tmp/{timestamp}', shell=True)
```

먼저 노트 내용을 `./tmp/notes.tmp`에 저장한 뒤, `cp` 명령어로 `/tmp/{timestamp}`에 복사한다.

문제는 `timestamp`가 사용자가 조작할 수 있는 쿠키 값인데도, `shell=True`로 실행하는 명령어 문자열에 직접 들어간다는 점이다. 쉘이 해당 문자열을 해석하므로 명령어 구분자 등 쉘 문법이 삽입되면 의도하지 않은 명령어가 실행될 수 있다.

`subprocess.Popen()`은 별도 프로세스를 실행하므로 명령어 출력이 Flask 응답에 그대로 표시되는 구조는 아니다.

**정리:** 노트 생성 → `backup-timestamp` 쿠키 설정 → 쿠키 값 변경 가능 → `/backup_notes` POST → `subprocess.Popen(..., shell=True)` 실행.

**취약점 원인:** 사용자 제어가 가능한 `backup-timestamp` 쿠키를 충분히 검증하지 않고 `shell=True` 명령어에 삽입하여 OS Command Injection이 발생한다.
