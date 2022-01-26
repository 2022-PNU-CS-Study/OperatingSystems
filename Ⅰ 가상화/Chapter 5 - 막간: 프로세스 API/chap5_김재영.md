# 5장

### 프로세스의 생성과 소멸에는 어떤 도구들이 쓰일까?

## 프로세스 API

> 프로세스의 생성과 제어를 위해 운영체제는 우리에게 **인터페이스**를 제공합니다.
> 

### fork() 시스템 콜

fork() 는 동작이 조금 특이한데, 코드를 먼저 살펴보겠습니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (original process)
        printf("hello, I am parent of %d (pid:%d)\n",
	       rc, (int) getpid());
    }
    return 0;
}
```

```bash
❯ ./p1
hello world (pid:77430)
hello, I am parent of 77431 (pid:77430)
hello, I am child (pid:77431)
```

fork() 시스템 콜은 프로세스의 생성에 사용됩니다. 위 프로그램은 fork() 를 사용했고, 결과적으로 자신과 같은 프로세스가 하나 더 생성되었습니다. 하지만 특이한점은 결과화면을 보면 알 수 있듯이 main() 의 시작점인 printf(”hello world”) 가 한 번 밖에 실행되지 않았다는겁니다. fork() 로 생성된 자식 프로세스는 main() 의 다음줄이 아닌 **fork() 가 수행된 시점에서 시작**된거죠.

위 코드의 수행 결과 화면이 항상 같지는 않습니다. 이번의 경우 fork() 직후 부모 프로세스가 먼저 실행되었지만, 그 반대도 충분히 일어날 수 있습니다. 
CPU 스케줄러는 굉장히 복잡하고 상황에 따라 유동적으로 다른 선택을 하기 때문에 **어느 프로세스가 먼저 실행되리라 단정하기는 힘듭니다.**

이와 같은 문제를 비결정성(nondeterminism)이라 부르며, 이로 인해 멀티 스레드 프로그램을 실행할 때 수많은 문제가 발생합니다. → 2장 병행성에서 자세하게 다룸

### wait() 시스템 콜

물론 위 프로그램이 항상 동일한 결과를 출력하도록 만들수도 있습니다. wait() 시스템 콜을 사용하여 항상 부모 프로세스가 자식 프로세스보다 늦게 출력하도록 만드는거죠.

wait() 은 해당 프로세스를 잠시 대기하도록 합니다. 하지만 마냥 대기만 할수는 없고 자식 프로세스가 종료할 때 까지만 기다립니다. 부모를 Blocked 상태로 만들고 자식 프로세스가 종료되면 Ready 상태로 돌립니다.

이렇게만 보면 wait() 은 그저 기다리는 동작만 하는 별거아닌 시스템 콜로 보일 수도 있지만 wait() 은 프로세스가 정상적으로 소멸하기 위해 굉장히 중요한 역할을 합니다. wait() 으로 **좀비 프로세스의 자원을 회수하고 고아 프로세스가 생기는 것을 방지**할 수 있습니다. 자식 프로세스가 실행중이라면 대기하고 이미 종료되었다면 즉시 반환하도록 동작합니다.

모든 프로세스는 종료 후 좀비 상태를 거칩니다. 하지만 부모 프로세스가 wait() 으로 자원을 회수하는 과정이 굉장히 빠르게 일어나기 때문에 눈에 잘 띄지 않을 뿐이죠. 

그리고 부모 프로세스가 자식 프로세스보다 먼저 종료되는 경우 자식 프로세스는 부모를 잃었기 때문에 고아 프로세스(orphan process)라 불립니다. 운영체제는 당연히 이런 고아 프로세스를 가만히 두지 않습니다. PID 1번인 init 프로세스가 고아 프로세스의 부모 역할을 대신하도록합니다.(입양) 없어진 부모 프로세스 대신 wait() 시스템 콜을 수행해서 고아 프로세스가 정상적으로 종료할 수 있게 해줍니다. 
아무리 init 프로세스가 고아원 역할을 해준다 한들 이것만 믿고 고아 프로세스를 마구 생성해서는 안됩니다. (성능저하를 일으킬 수 있음) 

### exec() 시스템 콜

exec() 시스템 콜은 fork() 처럼 프로세스를 생성하는 동작을 하지만 **자신이 아닌 다른 프로그램을 실행**할 때 쓴다는 점에서 차이가 있습니다. exec() 도 동작 방식이 특이한데, 실행할 파일(혹은 명령어)를 첫번째 인자(e.g. `ls`)로 받고 그와 관련해서 같이 쓰일 옵션값들(`-al`) 또한 뒤이어 인자로 받아서 **exec() 시스템 콜을 호출한 현재 프로세스의 코드 세그먼트와 정적 데이터 부분을 덮어씌워버립니다. 별도의 공간을 따로 생성하지 않는겁니다.**  따라서 exec() 를 호출한 프로세스는 온데간데없고 exec() 에 의해 호출된 프로세스만 메모리에 남게 되는거죠. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
        char *myargs[3];
        myargs[0] = strdup("wc");   // program: "wc" (word count)
        myargs[1] = strdup("p3.c"); // argument: file to count
        myargs[2] = NULL;           // marks end of array
        execvp(myargs[0], myargs);  // runs word count
        printf("this shouldn't print out");
    } else {
        // parent goes down this path (original process)
        int wc = wait(NULL);
        printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",
	       rc, wc, (int) getpid());
    }
    return 0;
}
```

```bash
❯ ./p3
hello world (pid:90660)
hello, I am child (pid:90661)
      32     123     966 p3.c
hello, I am parent of 90661 (wc:90661) (pid:90660)
```

위 코드에서 자식 프로세스가 생성된 이후의 코드를 보면(else if 이후) exec() 시스템 콜의 한 종류인 execvp() 시스템 콜로 인자들을 전달하고 실행하는 것을 볼 수 있습니다. 배열로 묶어서 쓰니 복잡해 보이는데 한줄로 풀어쓰면 다음과 같습니다. 

```c
execvp(”wc”, “p3.c”, NULL) 
```

위 한 줄의 코드 이후 현재 프로세스를 `wc p3.c` 라는 명령어로 실행되는 프로그램으로 완전히 대체하게 됩니다. execvp() 이후의 코드는 실행되지 않습니다. 해당 코드는 이미 사라지고 없습니다.

exec() 는 해당 프로세스를 대체하기 때문에 호출한 프로세스의 PID 또한 그대로 가져갑니다. 

정리하면 fork() 는 자신과 같은 프로세스를 하나 더 만들고 exec() 는 자신을 새로운 프로세스로 덮어 씌웁니다.

그리고 정확히 말하면 exec() 라는 하나의 시스템 콜은 존재하지 않습니다. exec() 는 공통된 작업을 수행하는 family 명칭이며 세부적인 구현과 사용 방법의 차이에 따라 여러 이름이 존재합니다.

### fork() 와 exec() 가 분리된 이유

새로운 작업을 실행하려면 fork() 와 exec() 는 항상 같이 쓰이는 것 같은데 이 둘을 왜 굳이 분리했을까요? 생각해보면 fork() 와 exec() 가 항상 연속적으로 실행되지는 않습니다. 운영체제가 여러 프로세스를 연속적으로 실행하는 동안 각 프로세스 마다 다음 프로세스를 위한 준비 동작이 필요한 경우가 있습니다. 이 동작은 fork() 와 exec() 사이에 수행할 수 있는거죠.

---

[kernel of linux - system call(2)](https://wogh8732.tistory.com/282)

[fork()](https://blog.potados.com/dev/things-happend-after-fork/)

[fork()와 vfork()의 차이점과 COW(Copy On Write) - Kernel - iamroot.org](http://www.iamroot.org/xe/index.php?mid=Kernel&document_srl=23297)

[Chapter 0 - Operating System Interfaces](https://92-lwj.tistory.com/4)

[멀티 쓰레드 환경에서 fork는 조심해야 한다.](https://blog.seulgi.kim/2016/03/fork-in-multithread.html)
