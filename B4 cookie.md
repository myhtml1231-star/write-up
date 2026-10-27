<img width="951" height="168" alt="image" src="https://github.com/user-attachments/assets/23f870a5-064e-49df-82d4-012af542bdb1" />


    username = request.cookies.get('username', None)
    if username:
        return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not admin"}')

        여기서 헤더만 검사하므로 헤더 값을 admin으로 바꾼다

        #!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

users = {
    'guest': 'guest',
    'admin': FLAG
}

@app.route('/')
def index():
        username = request.cookies.get('username', None)
        if username:
            return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not admin"}')
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        try:
            pw = users[username]
        except:
            return '<script>alert("not found user");history.go(-1);</script>'
        if pw == password:
            resp = make_response(redirect(url_for('index')) )
            resp.set_cookie('username', username)
            return resp 
        return '<script>alert("wrong password");history.go(-1);</script>'

app.run(host='0.0.0.0', port=8000)


guest로 로그하고 개발자 도구를 켜서 확인한다


<img width="679" height="303" alt="image" src="https://github.com/user-attachments/assets/2b6df182-ebf8-4abe-8e9d-58027fe37d1d" />

guest로 로그인된걸 확인하고

<img width="958" height="302" alt="image" src="https://github.com/user-attachments/assets/48a15300-ffcb-485c-87fe-19475c6721c4" />

쿠키 값을 admin으로 바꾼다

<img width="590" height="69" alt="image" src="https://github.com/user-attachments/assets/fc4da2b7-cd54-4b5f-ad61-a1277bb7a6b6" />


<img width="1032" height="239" alt="image" src="https://github.com/user-attachments/assets/fd690dd7-7c0b-4d19-8627-ab74292eb946" />

플래그 값을 확인한다


