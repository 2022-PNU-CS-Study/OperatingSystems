# 프로세스 API
> 회사는 사용할 수 없는 지식을 가진 사람을 채용하지 않는다. 🔥 🥲
---
## fork()
프로세스 생성에 사용
### 코드
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    printf("hello world (pid:%d)\n", (int)getpid());
    int rc = fork();
    if(rc < 0){ // fork 실패
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if(rc == 0) { // 자식 프로세스
        printf("hello, I am a child (pid:%d)\n", (int)getpid());
    } else {    // 부모 프로세스
        printf("hello, I am parent of %d (pid:%d)\n", rc, (int)getpid());
    }
    return 0;
}
```
### 출력
```shell
prompt> ./p1
hello world (pid:29146)
hello, I am parent of 29147 (pid:29146)
hello, I am child (pid:29147)
```
### 순서
1. ./p1 실행
2. hello world (pid:29146)출력
3. fork()로 자식 프로세스 생성, **이때 자식 프로세스는 부모 프로세스의 복사본이다.**
4. 부모 프로세스 실행 또는 자식 프로세스 실행
   - 단일 CPU 환경에서는 상황에 따라서 먼저 실행되는 프로세스가 다르다.
   - CPU의 스케줄러가 실행할 프로세스를 선택한다. 이러한 비결정성(nondeterminism)때문에 멀티 스레드환경에서 다양한 문제점이 발생한다.
### 설명
- 자식 프로세스는 부모 프로세스를 복사하여 실행하지만 처음부터 실행되는 것이 나닌 fork() 이후 부터 실행된다.(hello world가 한번만 출력된 이유)
- 자식 프로세스는 자신의 주소 공간, 레지스터, PC 값을 가진다. 
- fork()함수를 실행하면 부모 프로세스는 생성된 자식의 pid를 반환 받고 자식 프로세스는 0을 반환받는다.
## wait()
부모 프로세스가 자식 프로세스를 기다리기 위해서 사용된다. wait()를 사용하면 지정해준 순서대로 프로세스를 실행하는 것이 가능하다.

waitpid()는 더 많은 기능을 지원한다.
### 코드
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wiat.h>

int main(int argc, char* argv[])
{
    printf("hello world (pid:%d)\n", (int)getpid());
    int rc = fork();
    if(rc < 0){ // fork 실패
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if(rc == 0) { // 자식 프로세스
        printf("hello, I am a child (pid:%d)\n", (int)getpid());
    } else { // 부모 프로세스
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int)getpid());
    }
    return 0;
}
```
### 출력
```shell
prompt> ./p1
hello world (pid:29266)
hello, I am child (pid:29267)
hello, I am parent of 29267 (rc_wait:29267) (pid:29266)
```
## exec()
자기 자신이 아닌 다른 프로그램을 실행해야할 때 사용한다. 

exec()는 시행 파일의 이름과 필요한 인자가 주어지면 해당 실행 파일의 코드와 정적 데이터를 읽어들여 현재 실행중인 프로세스의 코드 세그먼트와 정적 데이터 부분을 덮어쓴다. 힙과 스택 및 프로그램 다른 주소 공간들로 새로운 프로그램의 실행을 위해 다시 초기화된다. 현재 실행중인 프로그램을 다른 프로그램으로 대체하는 것이다. 
### 코드
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wiat.h>

int main(int argc, char* argv[])
{
    printf("hello world (pid:%d)\n", (int)getpid());
    int rc = fork();
    if(rc < 0){ // fork 실패
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if(rc == 0) { // 자식 프로세스
        printf("hello, I am a child (pid:%d)\n", (int)getpid());
        char *myargs[3];
        myargs[0] = strdup("wc");   // 실행할 프로그램 이름
        myargs[1] = strdup("p3.c"); // 인자
        myargs[2] = NULL;           // 배열의 끝 표시
        execvp(myargs[0], myargs);  // "wc" 실행
        pringf("this shouldn't print out");
    } else {
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int)getpid());
    }
    return 0;
}
```
### 실행
1. 부모 프로세스 실행
2. exec()로 wc 실행
3. 부모 프로세스는 wait()로 자식 프로세스 기다림
4. wc 실행 후 부모 프로세스 실행