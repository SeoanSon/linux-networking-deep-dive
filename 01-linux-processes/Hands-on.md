# 01: Linux Processes 실습

## 🎯 실습 목표
- 프로세스 생성 및 관리
- 메모리 구조 이해
- IPC 메커니즘 실제 사용

## 📋 준비사항
- Linux 시스템 (WSL2, Ubuntu, 또는 Docker)
- gcc 컴파일러
- strace, gdb 등 디버깅 도구

## 🔬 실습 1: 프로세스 기본

### 1.1 프로세스 정보 확인

```bash
# 현재 프로세스 확인
ps aux

# 트리 구조로 보기
ps -ef --forest

# PID 추적
ps -p <PID> -o pid,ppid,cmd

# 메모리 사용 확인
ps aux | grep <프로세스명>
```

### 1.2 프로세스 생성

```bash
# Bash 스크립트로 자식 프로세스 생성
#!/bin/bash
echo "부모 PID: $$"
echo "자식 프로세스 시작"
sleep 10 &
CHILD_PID=$!
echo "자식 PID: $CHILD_PID"
ps -p $CHILD_PID -o ppid
```

## 🔬 실습 2: 메모리 구조

### 2.1 프로세스 메모리 맵 확인

```bash
cat /proc/<PID>/maps

# 예시 출력:
# 7f1234560000-7f1234561000 r--p ... /lib/x86_64-linux-gnu/libc.so.6
# 7fff12345000-7fff12346000 rw-p [stack]
# ffffffffff600000-ffffffffff601000 r-xp [vsyscall]
```

### 2.2 C 프로그램으로 메모리 확인

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

int global_data = 100;          // Data segment
static int static_data = 50;    // BSS segment (또는 Data)

void stack_demo() {
    int local_var = 42;         // Stack
    char *heap_ptr = malloc(50); // Heap
    
    printf("PID: %d\n", getpid());
    printf("&global_data (Data): %p\n", (void*)&global_data);
    printf("&static_data (BSS): %p\n", (void*)&static_data);
    printf("&local_var (Stack): %p\n", (void*)&local_var);
    printf("heap_ptr (Heap): %p\n", (void*)heap_ptr);
    printf("\n메모리 맵 확인:\n");
    printf("cat /proc/%d/maps\n", getpid());
    
    getchar();  // 일단 대기
    free(heap_ptr);
}

int main() {
    stack_demo();
    return 0;
}
```

컴파일 및 실행:
```bash
gcc -o memory_demo memory_demo.c
./memory_demo

# 다른 터미널에서:
cat /proc/$(pgrep memory_demo)/maps
```

## 🔬 실습 3: IPC (Inter-Process Communication)

### 3.1 Pipe 사용

```bash
# 명령어로 파이프 테스트
echo "Hello" | cat
ps aux | grep bash

# 또는 프로그램으로
#!/bin/bash
# write_to_pipe.sh
echo "메시지" > /tmp/mypipe &
# 다른 터미널:
# cat /tmp/mypipe
```

### 3.2 Named Pipe (FIFO)

```bash
# FIFO 생성
mkfifo /tmp/mypipe

# 터미널 1: 읽기 대기
cat /tmp/mypipe

# 터미널 2: 쓰기
echo "Hello from FIFO" > /tmp/mypipe

# 정리
rm /tmp/mypipe
```

### 3.3 Socket 통신

```bash
# 간단한 Socket 서버/클라이언트
# server.c
#include <stdio.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

int main() {
    int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = {};
    addr.sun_family = AF_UNIX;
    strcpy(addr.sun_path, "/tmp/socket.sock");
    
    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 1);
    
    int client_fd = accept(server_fd, NULL, NULL);
    char buf[256];
    read(client_fd, buf, sizeof(buf));
    printf("수신: %s\n", buf);
    
    close(client_fd);
    close(server_fd);
    unlink("/tmp/socket.sock");
    return 0;
}
```

## 🔬 실습 4: 신호 (Signals)

### 4.1 신호 핸들링

```bash
#!/bin/bash
# signal_demo.sh
trap 'echo "SIGINT 수신!"; exit' INT
trap 'echo "SIGTERM 수신!"; exit' TERM

echo "PID: $$"
echo "Ctrl+C 또는 kill $$를 입력하세요"
while true; do
    sleep 1
done
```

실행:
```bash
bash signal_demo.sh
# 다른 터미널에서:
kill -SIGTERM <PID>
```

### 4.2 신호 보내기

```bash
# 프로세스에 신호 전송
kill -SIGTERM <PID>
kill -9 <PID>  # SIGKILL (강제 종료)

# 모든 프로세스에 신호
killall <프로세스명>
```

## 🔬 실습 5: 프로세스 상태 추적

### 5.1 strace로 시스템 호출 추적

```bash
# 모든 시스템 호출 보기
strace ls -la

# 특정 시스템 호출만
strace -e open,read,write ls

# 실행 중인 프로세스 추적
strace -p <PID>
```

### 5.2 gdb로 디버깅

```bash
gcc -g -o myapp myapp.c
gdb ./myapp

# gdb 명령어
(gdb) run          # 실행
(gdb) break main   # 중단점
(gdb) step         # 한 줄 실행
(gdb) print var    # 변수 확인
(gdb) backtrace    # 호출 스택
```

## 📊 실습 결과 확인

```bash
# 프로세스 트리
pstree -p

# 프로세스 자원 사용
top -b -p <PID>

# 메모리 상세 정보
cat /proc/<PID>/status
```

---

**다음**: Linux Network Namespace 실습 🔗
