# B2 · addition-quiz

### 1. 문제 분석

프로그램은 난수 두 개를 더하는 문제를 50번 출제한다.

```c
for (int i = 0; i < 50; i++) {
    alarm(1);
    num1 = rand() % 10000;
    num2 = rand() % 10000;
    printf("%d+%d=?\n", num1, num2);
    scanf("%d", &inpt);

    if (inpt != num1 + num2) {
        printf("Wrong...\n");
        return 0;
    }
}
puts("Nice!");
puts(flag);
```

각 문제에서 `alarm(1)`이 1초 제한을 설정한다. 답이 틀리면 종료되고, **50문제를 모두 맞히면** FLAG가 출력된다. 매번 출력되는 숫자를 읽어 계산하면 되므로 난수를 예측할 필요는 없다.

### 2. 풀이 스크립트

```python
from pwn import *

r = remote('host3.dreamhack.games', 14005)

for i in range(50):
    first_number = int(r.recvuntil(b'+', drop=True))
    second_number = int(r.recvuntil(b'=?\n', drop=True))

    r.sendline(str(first_number + second_number).encode())

r.interactive()
```

`remote()`는 문제 서버의 주소와 포트에 연결한다. 위 주소와 포트는 풀이 당시 값이므로 재실행할 때는 문제 페이지에서 발급받은 값으로 바꾼다.

### 3. 통신 흐름과 반복문

서버는 한 문제를 보내고 `scanf()`로 답을 기다린다. 정답을 받아야 다음 문제를 보내므로, 같은 연결에서 **수신 → 숫자 추출 → 계산 → 답 전송**을 50번 반복해야 한다. 1초 제한 때문에 이 과정을 스크립트로 자동화했다.

예를 들어 서버가 `2917+9658=?\n`을 보내면:

1. `recvuntil(b'+', drop=True)`가 `+`까지 읽고 구분자를 제외한 `b'2917'`을 반환한다.
2. `recvuntil(b'=?\n', drop=True)`가 이어서 `=?\n`까지 읽고 구분자를 제외한 `b'9658'`을 반환한다.
3. `int()`로 두 바이트 값을 정수로 바꿔 더한다.
4. `str(...).encode()`로 합계를 문자열을 거쳐 바이트로 바꾸고, `sendline()`이 줄바꿈을 붙여 `b'12575\n'`을 전송한다.

`recvuntil()`은 지정한 구분자가 도착할 때까지 데이터를 읽는다. `b'...'`는 통신에 사용하는 바이트 자료형이고, `drop=True`는 반환값에서 구분자를 제거한다. 한 번에 문제 전체가 도착하지 않아도 구분자를 기준으로 필요한 부분을 읽을 수 있다.

### 4. 결과 확인

반복문이 끝난 후 `interactive()`로 연결을 대화형 모드로 전환해 서버의 남은 출력을 확인한다. 실행 결과 `Nice!`와 FLAG가 출력되었다.

```text
Nice!
DH{0472d70efbc7fee1de726614d534661e0858541bf26ab992a408de549518ccaf}
```

이후 표시된 `EOF`는 프로그램이 출력을 마치고 연결을 종료했다는 뜻이다.
