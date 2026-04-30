Brand new pintos for Operating Systems and Lab (CS330), KAIST, by Youngjin Kwon.

The manual is available at https://casys-kaist.github.io/pintos-kaist/.


# 🖥️ PintOS — Kaist OS Implementation

> Stanford PintOS를 기반으로 한 교육용 운영체제 구현 프로젝트  
> **Branch:** `develop/songarden` | **Author:** `songarden` (Son Jung Won)  
> **Period:** 2023.11 ~ 2023.12

---

## 📋 목차

- [프로젝트 개요](#-프로젝트-개요)
- [구현 범위](#-구현-범위)
- [Project 1 — Threads](#-project-1--threads)
- [Project 2 — User Programs](#-project-2--user-programs)
- [Project 3 — Virtual Memory](#-project-3--virtual-memory)
- [커밋 히스토리 요약](#-커밋-히스토리-요약)
- [디렉터리 구조](#-디렉터리-구조)

---

## 🧭 프로젝트 개요

PintOS는 x86-64 아키텍처 기반의 교육용 운영체제 프레임워크입니다. 이 저장소는 카이스트 운영체제 수업의 과제로 수행한 구현 결과물을 담고 있으며, 총 3개의 프로젝트(Threads / User Programs / Virtual Memory)에 걸쳐 핵심 OS 컴포넌트를 직접 구현하였습니다.

| 항목 | 내용 |
|------|------|
| 기반 프레임워크 | Stanford PintOS (x86-64) |
| 구현 언어 | C |
| 총 커밋 수 | 87 commits |
| 개발 기간 | 2023.11.27 ~ 2023.12.29 |

---

## 🗂️ 구현 범위
```
✅ Project 1 - Threads
├── Alarm Clock (sleep/wake 메커니즘)
├── Priority Scheduling
├── Priority Donation
└── Advanced Scheduler (MLFQS)
✅ Project 2 - User Programs
├── Argument Passing
├── System Call (halt, exit, exec, wait, fork)
├── File System Calls (open, close, read, write, create, remove, ...)
└── dup2 (Extra)
✅ Project 3 - Virtual Memory
├── Supplemental Page Table (SPT)
├── Lazy Loading
├── Stack Growth
├── Memory-Mapped Files (mmap / munmap)
└── Swap In & Out (Clock Eviction Policy)
```
---

## 🧵 Project 1 — Threads

### 1-1. Alarm Clock

기존의 busy-waiting 방식 대신, **sleep/wake 리스트 기반의 효율적인 sleep 구현**으로 교체하였습니다.

- `thread_sleep_and_yield()` 함수를 통해 현재 스레드를 `sleep_list`에 삽입 후 블록
- `sleep_list`는 **wake-up 시각 기준 오름차순 정렬** 유지 → 최소 시각만 검사하는 최적화
- 타이머 인터럽트에서 `time_to_wake()` 검사를 통해 깨워야 할 스레드를 `thread_unblock()`
- idle thread가 ready_list에 들어가지 않도록 예외 처리
```
thread_sleep_and_yield()
time_to_wake()
thread_less()         ← 우선순위 비교 함수 (sleep_list 정렬용)
```
### 1-2. Priority Scheduling

**우선순위 기반 스케줄링**을 구현하여 높은 우선순위 스레드가 항상 먼저 실행되도록 하였습니다.

- `ready_list`를 우선순위 내림차순으로 항상 정렬 유지
- 새 스레드 생성/우선순위 변경 시 즉시 `thread_yield()` 호출하여 선점
- `semaphore`, `condition variable`의 waiters 리스트도 우선순위 기준 정렬
```
check_running_priority()    ← 현재 실행 스레드보다 높은 우선순위 스레드 존재 시 yield
list_sort_high_priority()   ← ready_list 우선순위 정렬
thread_more_priority()      ← 비교 함수
```
### 1-3. Priority Donation

**중첩 우선순위 기부(Nested Donation)** 와 **다중 기부(Multiple Donation)** 를 모두 지원합니다.

- `lock_acquire()` 시 현재 스레드의 우선순위를 lock holder에게 기부
- `wait_to_lock` 포인터를 통해 체인 구조로 연결된 락에 재귀적으로 기부
- `lock_release()` 시 해당 락에 물려 있던 기부 우선순위를 회수하고, 나머지 기부 값 중 최대값으로 복원
- `thread_mlfqs` 플래그를 통해 MLFQS 모드에서는 기부 비활성화
```
thread_more_lock_priority()   ← lock 기반 우선순위 비교
```
### 1-4. Advanced Scheduler (MLFQS)

**BSD 스케줄러 방식의 MLFQS**를 고정소수점 연산을 활용하여 구현하였습니다.

- 매 틱마다 현재 스레드의 `recent_cpu` += 1
- 매 초(TIMER_FREQ)마다 전체 스레드의 `recent_cpu`와 `priority` 재계산
- 전역 `load_avg` 값을 갱신하여 시스템 부하를 반영
- 64개의 우선순위 큐를 통해 스케줄링
```
set_load_avg()
set_recent_cpu_and_priority()
thread_set_mlfqs_priority()
fp_multiple(), fp_divide()
fp_to_int_round(), fp_to_int_toward_zero()
```
---

## 💻 Project 2 — User Programs

### 2-1. Argument Passing

유저 프로그램 실행 시 명령줄 인자를 올바르게 스택에 세팅합니다.

- `parsing_file_input()` 함수를 구현하여 인자를 **오른쪽 → 왼쪽 순**으로 스택에 push
- Word alignment, argv 포인터 배열, argc, 리턴 주소 순서를 x86-64 calling convention에 맞게 배치
- `hex_dump()`를 통해 스택 레이아웃 검증 완료
- `process_create_initd()`에서 파싱하여 파일 이름만 스레드 이름으로 전달

### 2-2. System Calls

사용자 공간과 커널 공간을 잇는 **시스템 콜 인터페이스** 전반을 구현하였습니다.

#### 프로세스 관련

| 시스템 콜 | 구현 내용 |
|-----------|-----------|
| `halt` | `power_off()`로 OS 종료 |
| `exit` | exit_status 저장 후 프로세스 종료, 부모에게 상태 전달 |
| `exec` | `process_exec()` 호출 (복사본으로 실행) |
| `fork` | `__do_fork()` + `duplicate_pte()`로 자식 프로세스 생성 |
| `wait` | 세마포어(`child_wait_sema`)를 통한 자식 종료 대기 |

**fork/wait 동기화 설계:**
- `exit_sema`: 자식이 종료될 때 부모가 exit_status를 읽기 전까지 자식 메모리를 보존
- `child_wait_sema`: 부모가 자식의 종료를 기다리는 대기 세마포어
- `parent` 멤버를 통해 자식이 부모에게 직접 exit_status 전달

#### 파일 시스템 관련

| 시스템 콜 | 구현 내용 |
|-----------|-----------|
| `open` | 파일 열기 + 파일 디스크립터 테이블(fdt)에 등록 |
| `close` | fdt 해제, 로딩 중인 파일 보호 |
| `read` | stdin(fd=0) 또는 파일에서 읽기 |
| `write` | stdout(fd=1) 또는 파일에 쓰기 |
| `create` | 파일 생성 |
| `remove` | 파일 삭제 |
| `filesize` | 파일 크기 반환 |
| `seek` / `tell` | 파일 포지션 제어 |

**파일 디스크립터 설계:**
- `thread` 구조체에 `fdt` (File Descriptor Table) 멤버 추가
- 0번(stdin), 1번(stdout) 사전 예약
- `process_add_fd()`로 새 파일 등록, 최대 FD 개수 관리
- `filesys_lock`을 통한 파일 시스템 동시 접근 제어

### 2-3. dup2 (Extra)

`dup2` 시스템 콜을 구현하여 파일 디스크립터 복제를 지원합니다.

- `thread` 구조체에 `dup_table` 및 `dup_max` 멤버 추가
- `close()` 시 dup 테이블을 순회하여 복제된 fd가 있으면 파일을 닫지 않고 dup 엔트리만 제거
- `process_exit()` 시 dup 테이블 내 fd 존재 여부 확인 후 안전하게 정리

---

## 🧠 Project 3 — Virtual Memory

### 3-1. Supplemental Page Table (SPT)

페이지 폴트 처리와 페이지 관리를 위한 **보조 페이지 테이블**을 해시 테이블로 구현하였습니다.

- `page` 구조체: 가상 주소(`va`), 페이지 타입(`VM_ANON` / `VM_FILE` / `VM_UNINIT`), writable 플래그, 프레임 포인터 포함
- `spt_find_page()`, `spt_insert_page()`, `spt_remove_page()` 구현
- SPT 수정/삭제 연산에 세마포어를 적용하여 race condition 방지
- `supplemental_page_table_kill()`: `hash_clear`로 내부 페이지 free, `supplemental_page_table_destroy()`에서 hash 자체 해제
```
spt_find_page()
spt_insert_page()
spt_remove_page()
supplemental_page_table_init()
supplemental_page_table_copy()
supplemental_page_table_kill()
hash_action_free()
```
### 3-2. Lazy Loading

ELF 세그먼트를 **실제 접근 시점에 물리 프레임을 할당**하는 지연 로딩을 구현하였습니다.

- `load_segment()`에서 각 페이지를 즉시 로드하지 않고 `vm_alloc_page_with_initializer()`로 SPT에만 등록
- `VM_UNINIT` 타입으로 등록 후, 최초 접근 시 page fault → `lazy_load_segment()` 콜백 실행
- `vm_alloc_page_with_initializer()`에서 `writable` 값이 덮어씌워지는 버그 수정 (`uninit_new` 이후 값 대입)
- `setup_stack`은 즉시 물리 프레임 할당 (lazy loading 예외)
```
lazy_load_segment()          ← page fault 시 실제 파일 데이터 로드
vm_alloc_page_with_initializer()
vm_try_handle_fault()
vm_do_claim_page()
vm_connect_page_frame()
```
### 3-3. Stack Growth

유저 스택이 필요에 따라 자동으로 확장되는 **스택 성장** 기능을 구현하였습니다.

- page fault 발생 시 접근 주소가 `rsp - 8` 이하이고 스택 한계 내에 있을 경우 `vm_stack_growth()` 호출
- `setup_stack` 시 런타임 스택임을 나타내는 `marker_0` 타입 플래그 추가
- `check_addr`에서 VM 모드에서는 pml4 조회 실패 시 바로 exit하지 않고 page fault로 전달
```
vm_stack_growth()
vm_try_handle_fault()        ← 스택 성장 조건 판단
```
### 3-4. Fork with VM

VM 환경에서의 **프로세스 복제(fork)**를 안전하게 구현하였습니다.

- `supplemental_page_table_copy()`: 부모의 SPT를 순회하며 자식의 SPT에 페이지 복사
- `uninit_duplicate_aux()`: 아직 초기화되지 않은 페이지의 aux 데이터도 복제
- Copy-on-Write 미구현 대신 모든 페이지를 즉시 복사

### 3-5. Memory-Mapped Files (mmap / munmap)

파일을 유저 가상 주소 공간에 **메모리 매핑**하는 기능을 구현하였습니다.

- `do_mmap()`: 파일의 각 페이지를 `VM_FILE` 타입으로 SPT에 등록, lazy loading 방식 적용
- `lazy_load_file_segment()`: 실제 접근 시 파일에서 데이터 로드
- `do_munmap()`: 더티 페이지는 파일에 write-back 후 매핑 해제
- `file_backed_swap_in()` / `file_backed_swap_out()`: 파일 기반 페이지의 swap 처리
```
do_mmap()
do_munmap()
lazy_load_file_segment()
file_backed_swap_in()
file_backed_swap_out()
file_backed_destroy()
```
### 3-6. Swap In & Out (Clock Eviction)

물리 메모리가 부족할 때 **클락 알고리즘 기반의 페이지 교체**를 구현하였습니다.

- `frame_table`: 전역 프레임 테이블로 클락 알고리즘의 순환 포인터 관리
- `vm_get_victim()`: 클락 알고리즘으로 교체 대상 프레임 선정 (access bit = 0 → evict)
- `clock_evict_policy()`: access bit 초기화 → 클락 포인터 이동
- **익명 페이지(anon)**: 스왑 디스크에 swap_out, 필요 시 swap_in
- **파일 기반 페이지(file)**: 더티 페이지는 파일에 write-back
- `swap_lock`이 중복 acquire 문제를 일으켜 **세마포어로 교체**하는 버그픽스 수행
```
vm_get_frame()
vm_evict_frame()
vm_get_victim()
clock_evict_policy()
anon_swap_in()
anon_swap_out()
vm_remove_frame()
```
---

## 📝 커밋 히스토리 요약

| 날짜 | 주요 작업 |
|------|-----------|
| 2023.11.27 | sleep_list 정렬, ready_list 우선순위 정렬 구현 |
| 2023.11.28 | Priority Donation 구현, 세마포어 블록 시 예외 처리 |
| 2023.12.02 | MLFQS 완성 |
| 2023.12.04 | idle_thread ready_list 예외 처리 |
| 2023.12.05~08 | 인자 파싱, 시스템 콜(open/exec/exit/halt/fork/create) 구현 시작 |
| 2023.12.11~13 | fork/wait, filesize/read/seek/tell/close, dup2 구현 완성 |
| 2023.12.16 | 디버거 설정, idle_thread exit_sema 예외 처리 |
| 2023.12.19~21 | SPT 구현, lazy loading 구현 (실패 후 재시도 → 성공) |
| 2023.12.22~23 | VM fork/spt kill, stack growth, uninit duplicate_aux 구현 |
| 2023.12.24~25 | mmap/munmap 구현 및 버그 수정 |
| 2023.12.27~28 | 클락 기반 swap policy, file/anon 페이지 swap in&out 구현 |
| 2023.12.29 | swap lock → semaphore 교체 버그픽스, 주석 정리 |

---

## 📁 디렉터리 구조
```
pintos/
├── threads/          # Project 1: 스레드, 스케줄러, 동기화
│   ├── thread.c/.h   # 스레드 생성/스케줄링/MLFQS
│   └── synch.c/.h    # 세마포어/락/조건변수 + Priority Donation
├── userprog/         # Project 2: 유저 프로그램
│   ├── process.c/.h  # 프로세스 생성/종료/fork/exec/wait
│   └── syscall.c/.h  # 시스템 콜 핸들러
├── vm/               # Project 3: 가상 메모리
│   ├── vm.c/.h       # SPT, 프레임 테이블, 페이지 폴트 처리
│   ├── anon.c/.h     # 익명 페이지 (swap)
│   └── file.c/.h     # 파일 기반 페이지 (mmap)
├── include/          # 헤더 파일들
├── devices/          # 타이머, 디스크 등 디바이스
├── filesys/          # 파일 시스템 (기본 제공)
├── lib/              # 표준 라이브러리
└── tests/            # 테스트 케이스
```
---

## 🛠️ 개발 환경

- **OS:** Ubuntu (WSL 또는 Native Linux)
- **Architecture:** x86-64
- **Debugger:** GDB (`.vscode` 디버그 설정 포함)
- **Test:** `pintos --` 명령어 기반 통합 테스트

---

> *"Let's hit the pint Operating System"*
