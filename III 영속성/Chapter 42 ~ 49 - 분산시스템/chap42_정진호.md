# 분산 시스템

> 분산 시스템의 핵심은 실패와 고장의 극복이다.

## 신뢰할 수 없는 통신 계층

패킷 손실에 대응할 수 있는 좋은 방법 → 아무것도 안하기

현대의 모든 시스템에는 UDP/IP 네트워크 스택이 있다. UDP를 사용하기 위해서는 소켓 API를 이용하여 ::통신 지점(Communication end point)::을 생성한다. 연결된 프로세스들은 UDP 데이터그램(datagram)을 원래의 프로세스로 전송한다.

### 클라이언트-서버 UDP 통신

```other
// client
int main(int argc, char *argv[]){
	int sd = UDP_Open(20000);
	struct sockaddr_in addr, addr2;
	int rc = UDP_FillSockAddr(&addr, "sender", 10000);
	char message[BUFFER_SIZE];
	sprintf(message, "hello world");
	rc = UDP_Write(sd, &addr, message, BUFFER_SIZE);
	if(rc > 0){
		int rc = UDP_Read(sd, &addr2, buffer, BUFFER_SIZE);
	}
	return 0;
}
```

```other
// server
int main(int argc, char *argv[]){
	int sd = UDP_Open(10000);
	assert(sd > -1);
	while(1){
		struct sockaddr_in s;
		char buffer[BUFFER_SIZE];
		int rc = UDP_Read(sd, &s, buffer, BUFFER_SIZE);
		if(rc > 0){
			char reply[BUFFER_SIZE];
			sprintf(reply, "reply");
			rc = UDP_Write(sd, &s, reply, BUFFER_SIZE);
		}
	}
	return 0;
}
```

```other
int UDP_Open(int port){
	int sd;
	if((sd = socket(AF_INET, SOCK_DGRAM, 0)) == -1)
		return -1;
	struct sockaddr_in myaddr;
	bzero(&myaddr, sizeof(myaddr));
	myaddr.sin_family = AF_INET;
	myaddr.sin_port = htons(port);
	myaddr.sin_addr.s_addr = INADDR_ANY;
	if(bind(sd, (struct sockaddr *)&myaddr, sizeof(myaddr)) == -1) {
		close(sd);
		return -1;
	}
	return sd;
}
```

```other
int UDP_FillSockAddr(struct sockaddr_in *arrd, char *hostName, int port){
	bzero(addr, sizeof(struct sockaddr_in));
	addr->sin_family = AF_INET; // 호스트 바이트 오더
	addr->sin_port = htons(port); // short, 네트워크 바이트 오더
	struct in_addr *inAddr;
	struct hostent *hostEntry;
	if((hostEntry = gethostbyname(hostName)) == NULL)
		return -1;	
	inAddr = (struct in_addr*) hostEntry->h_addr;
	addr->sin_addr = *inAddr;
	return 0;
}

int UDP_Write(int sd, struct sockeaddr_in *addr, char *buffer, int n){
	int addrLen = sizeof(struct sockaddr_in);
	return sendto(sd, buffer, n, 0, (struct sockaddr*)addr, addrLen);
}

int UDP_Read(int sd, struct sockaddr_in *addr, char *buffer, int n){
	int len = sizeof(struct sockaddr_in);
	return recvfrom(sd, buffer, n, 0, (struct sockaddr*)addr, (socklen_t*) &len);
	return rc;
}
```

## 신뢰할 수 있는 통신 계층

신뢰할 수 있는 통신 계층을 만들기 위해서는 패킷 손실에 대응할 수 있는 메커니즘이 필요하다.

- 발신자는 수신자가 메시지를 수신했다는 것을 알아야한다.

   → 수신자가 발신자에가 ack 패킷을 전송해준다.

   - 발신자가 ack를 수신하지 못한 경우 타임아웃되어 재전송한다.
   - ack 패킷이 손실되어 송신자가 재전송하여도 데이터 패킷에는 순서가 있기 때문에 수신자는 같은 패킷 수신시에 다시 ack 를 보내준다.

## 통신 추상화

분산 시스템에서 하나의 시스템은 다른 시스템과 통신을 하는 것이 아닌 여러 쓰레드가 일하는 것처럼 추상화되어야한다.

## Remote Procedure Call(RPC)

RPC의 목표는 분산 시스템에서 코드 실행을 로컬 내의 함수를 부르는 것처럼 간단하고 복잡하지 않게 만드는 것이다. gRPC를 잠깐 사용해봤는데 gRPC는 외부 시스템 호출을 로컬 함수 사용하 듯이한다. RPC는 스텁 생성기(stub generator, = protocol compiler)와 런타임 라이브러리(run-time library)두 부분으로 나뉘어있다. 스텁 생성기를 통해서 우리는 인터페이스만 알면 생성된 스텁을 통해서 분산 시스템을 로컬함수처럼 사용할 수 있다.

### 스텁 생성기

함수의 인자들을 묶는 불편함을 없애고 자동으로 메시지를 만들어준다. 메시지는 분산 시스템간의 통신을 의미한다. 서버가 클라이언트에게 공지할 프로시저들의 집합을 컴파일러에 입력으로 전달한다.

```other
interface {
	int func1(int arg1);
	int func2(int arg1, int arg2);
};
```

~~근데 이거 운영체제랑 무슨 상관이지 앞부분 내가 안 읽었나 두괄식으로 해주세요~~

클라이언트 스텁을 사용해서 RPC 사용 과정

1. 메시지 버퍼 생성
2. 메시지 버퍼에 필요한 정보 직렬화
3. RPC 서버에 메시지 전송
4. 응답 대기
5. 리턴 코드와 인자 역직렬화
6. 호출자에게 리턴
7. 메시지 역직렬화
8. 실제 함수 호출
9. 결과 통합
10. 응답 전송

#### 스텁 컴파일러에서 고려 사항

1. 복잡한 구조의 인자나 다수의 인자를 전달해야한다.
2. 병행성을 고려하여 서버를 구성해야한다.
   - 쓰레드 풀을 사용해서 병행성 보장

### 런타임 라이브러리

성능과 신롸성 문제를 처리한다.

1. 원격 서비스의 위치를 찾아야한다.
   - 현재 인터넷 프로토콜이 사용하는 호스트의 이름과 포트번호를 사용한다.
   - 패킷이 특정 주소에서 시스템에 있는 다름 기계로 전달될 수 있는 메커니즘을 제공해야한다.
1. RPC를 어떤 전송 계층 프로코콜 위에 만들지 결정해야한다.
   - 신뢰할 수 있는 TCP/IP  vs 신뢰할 수 없는 UDP/IP
      - TCP는 느리기 때문에 많은 RPC는 UDP를 사용한다.
1. 원격 호출이 완료까지 오랜 시간이 걸리는 경우

   → 먼저 ack를 보낸 후 완료될 때까지 주기적으로 ack를 보낸다.

1. 한 패킷에 담을 수 있는 양보다 더 많은 수의 인자를 갖는 프로시저 호출들을 처리할 수 있어야한다.
   1. framgmentaion하고 reaaembly할 수 있어야한다.
2. 바이트 순서 표시(byte ordering)

# NFS(Network File System)

## 기본적인 분산 파일 시스템

클라이언트는 클라이언트 응용 프로그램이 클라이언트 측 파일 시스템을 통해 파일과 디렉터리를 접근한다. 그리고 클라이언트 프로그램이 서버에 있는 파일을 접근하려면 **클라이언트 측 파일 시스템** 에 시스템콜을 호출한다.분산 파일 시스템은 성능 부분을 제외하면 로컬 파일 시스템과 동일하게 접근할 수 있어야한다.

→ 분산 파일 시스템이지만 시스템콜을 호출하듯 사용할 수 있어야한다.

## 단순하고 빠른 서버 크래시 복구

NFS는 상태를 유지하지않는 stateless다. 클라이언트의 각 요청을 서버에서 독립적으로 수행된다.

클라이언트 측 파일 시스템이 파일을 열기 위해 서버에 파일 디스크립터를 요청한다. 파일 서버는 로컬 저장 장치에 있는 파일을 열어서 파일 디스크립터를 클라이언트에게 전달한다. 그 뒤로 해당 파일에 접근할 때는 클라이언트는 서버에 파일 디스크립터를 전달하여 요청한다. → 여기서 파일 디스크립터는 서버-클라이언트 사이의 공유되는 정보(shared state)이다.

클라이언트와 서버간 상태정보가 공유되면 크래시 과정이 복잡해진다. 두번째 요청을 전송하기 전에 서버기 크래시되면 서버는 공유된 정보를 가지고 있지 않아 클라이언트가 전달한 파일 디스크립터가 어떤 파일인지 알 수 없다.

→ 복구 프로코톨 실행(recovery protocol)

## NFSv2 프로토콜

- 파일 핸들(file handle): 특정 연산을 수행할 파일이나 디렉터리를 고유하게 설명하는데 사용 → 파일 디스크립터처럼 작용
- LookUp: 파일 핸들을 얻기 위해 사용.

## 분산 파일 시스템에 적용

![Image.png](https://res.craft.do/user/full/3442554f-f1f2-6ecd-471f-e2827e4b5905/doc/31E4C84A-6EEE-4AEE-892F-56A97D1F68F3/A73C14AF-0551-4C5A-95C1-494B0F40B16E_2/xU1VjyLvVe9bkIOaIGYtwAICyetdzQpLlQVRDmIOYh0z/Image.png)

1. 클라이언트가 파일 연산의 **상태** 를 관리하여 서버와 통신을 로컬 함수를 사용하듯 할 수 있다.
2. 서버는 디렉터리 또는 파일을 확인할 때 Lookup한다.
3. 서버에 요청을 보내면 해당 요청을 완료하는데 필요한 정보가 담겨있다.

## 멱등성을 사용한 에러 처리

중간에 패킷이 손실되면 재용청하면된다.

## 클라이언트 측 캐싱

캐싱을 사용하면 분산시스템의 단점인 지연시간을 어느정도 해결할 수 잇다. 하지만 여러개의 캐시를 사용하면 모든 캐시을 일관되게 유지해줘야한다.

# Andrew File System(AFS)

로컬 디스크에 파일 전체를 캐싱한다.

NFS는 새로운 파일 버전이 서버로 갱신되는지와 서버가 새로운 버전이 되었을 때 캐시도 갱신되어야한지가 문제였다.

AFS는 서버 내용이 변경되고 갱신된 파일이 닫힐 때 서버는 캐시된 사본을 가지고 있는 클라이언트들과 콜백을 모두 끊어서 클라이언트가 더이상 오래된 파일을 가지고 있지 않도록한다.

AFS는 일관성 유지를 위해서 마지막 기록자가 승리 방식을 하용한다. close()를 마지막으로 호출하는 클라이언트가 서버에 마지막으로 파일 전체를 갱신한다.

