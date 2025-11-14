# Basic Interpreter (`basic_interpreter.c`) 소스 코드 분석

> **OS 오전반 / 20210015 도현명**

## 1. 과제 목적

본 문서는 `basic_interpreter.c`로 작성된 C 기반 인터프리터의 소스 코드를 분석하는 것을 목적으로 한다. 프로그램의 전체 구조를 이해하고, 입력 파일(`.spl`)이 어떤 절차를 거쳐 최종 출력을 생성하는지 코드 레벨에서 추적한다.

이 문서는 핵심 자료구조, 주 실행 흐름, 함수 호출 관계를 다이어그램과 상세한 설명을 통해 기술한다.

<br>

## 2. 목차 (Table of Contents)

1. [과제 목적](#1-과제-목적)
2. [목차](#2-목차-table-of-contents)
3. [프로그램 전체 구조도](#3-프로그램-전체-구조도)
4. [핵심 자료구조](#4-핵심-자료구조)
   * [A. 메인 스택 (STACK)](#a-메인-스택-stack)
   * [B. 연산자 스택 (MathStack)](#b-연산자-스택-mathstack)
   * [C. 피연산자 스택 (CalcStack)](#c-피연산자-스택-calcstack)
5. [프로그램 전체 실행 흐름 (main 함수)](#5-프로그램-전체-실행-흐름-main-함수)
6. [핵심 기능 분석 (Line by Line)](#6-핵심-기능-분석-line-by-line)
   * [A. 스택 관리 (Push / Pop)](#a-스택-관리-push--pop)
   * [B. 수식 계산 (`(` 키워드 처리)](#b-수식-계산--키워드-처리)
   * [C. 변수/함수 값 조회 (GetVal)](#c-변수함수-값-조회-getval)
   * [D. 함수 호출 및 복귀 (Call & Return)](#d-함수-호출-및-복귀-call--return)
7. [Input1.spl 실행 추적 (Step-by-Step)](#7-input1spl-실행-추적-step-by-step)
   * [A. 실행 환경 및 결과](#a-실행-환경-및-결과)
   * [B. 1단계: `main` 함수 탐색 및 변수 정의](#b-1단계-main-함수-탐색-및-변수-정의)
   * [C. 2단계: `f(c)` 함수 호출](#c-2단계-fc-함수-호출)
   * [D. 3단계: `f` 함수 내부 실행 및 반환](#d-3단계-f-함수-내부-실행-및-반환)
   * [E. 4단계: `main` 함수 복귀 및 최종 연산](#e-4단계-main-함수-복귀-및-최종-연산)
   * [F. 5단계: 프로그램 종료](#f-5단계-프로그램-종료)
8. [결론](#8-결론)

<br>

## 3. 프로그램 전체 구조도

이 인터프리터는 `.spl` 파일을 입력받아, `main` 함수 내의 `while` 루프를 통해 한 줄씩 파싱하고 실행합니다. 3개의 핵심 스택을 사용하여 프로그램의 상태(STACK)와 수식 연산(MathStack, CalcStack)을 관리합니다.

> **[그림 제안 1: 전체 구조도]**
> 여기에 프로그램의 전체적인 데이터 흐름을 보여주는 다이어그램을 추가하세요.
>
> (예시: `input.spl 파일` -> `[ main() 함수 (fgets) ]` -> `[ 키워드 분석 ]` -> (분기)
> * (분기 1) `int`, `function` -> `[ STACK ]` (상태 저장)
> * (분기 2) `(` 수식 -> `[ MathStack ]` & `[ CalcStack ]` (연산)
> * (분기 3) `end` -> `[ STACK ]` (호출 스택 확인) -> `printf()` (출력) 또는 `filePtr` 되감기

<br>

## 4. 핵심 자료구조

이 인터프리터는 C의 `struct`와 포인터를 이용해 3가지 핵심 스택(Stack)을 구현하여 프로그램을 실행한다.

### A. 메인 스택 (STACK)

* **구조체:** `struct node` / `Stack`
* **용도:** 프로그램의 모든 '상태'를 저장하는 가장 중요한 스택. 변수, 함수 정의, 함수 호출 정보(복귀 주소) 등을 저장하여 **실행 스택(Call Stack)**과 **심볼 테이블(Symbol Table)**의 역할을 겸합니다.
* **저장 내용 (node.type):**
    * `type 1`: 변수 (Variable). (예: `int a=1`)
    * `type 2`: 함수 정의 (Function Definition). (예: `function f`)
    * `type 3`: 함수 호출 (Function Call). (예: `f(c)`. 복귀할 라인 저장)
    * `type 4`: 블록 시작 (`begin`).
    * `type 5`: 블록 종료 (`end`).
* **특징:** 변수의 스코프(Scope)와 함수 호출/복귀를 이 스택을 통해 관리한다.

```c
/* 1 var, 2 function, 3 function call, 4 begin, 5 end */
struct node {
    int type; 
    char exp_data;
    int val;
    int line;
    struct node* next;
};
```

### B. 연산자 스택 (MathStack)

* **구조체:** `struct opnode` / `OpStack` (코드에는 `opstack`으로 되어 있으나 편의상 `MathStack`으로 칭함)
* **용도:** 수식 계산 시, 중위 표기법(Infix)을 후위 표기법(Postfix)으로 변환하기 위해 연산자(+, -, *, /)를 임시 저장한다.

```c
struct opnode { 
    char op; 
    struct opnode* next; 
};
```

### C. 피연산자 스택 (CalcStack)

* **구조체:** `struct postfixnode` / `PostfixStack`
* **용도:** 후위 표기법(Postfix)으로 변환된 수식을 실제 계산하기 위해 피연산자(숫자)를 저장한다.

```c
struct postfixnode { 
    int val; 
    struct postfixnode* next; 
};
```

## 5. 프로그램 전체 실행 흐름 (main 함수)

`main` 함수는 인터프리터의 전체 실행을 주관한다.

1.  **초기화:** `MathStack`, `CalcStack`, `STACK` 3개의 스택을 위한 메모리를 할당한다.
2.  **인자 확인:** `argc != 2`인지 확인하여, 프로그램 실행 시 `.spl` 파일이 정확히 1개 주어졌는지 검사한다.
3.  **파일 열기:** `fopen(argv[1], "r")`로 입력 `.spl` 파일을 연다.
4.  **메인 파싱 루프:** `while (fgets(line, 4096, filePtr))`
    * 파일을 한 줄씩 읽어 `line` 버퍼에 저장한다.
    * `curLine`을 증가시켜 현재 라인 번호를 추적한다.
    * `rstrip()`으로 줄 끝의 공백/개행 문자를 제거한다.
    * `my_stricmp()`(대소문자 무시 문자열 비교)를 사용해 키워드를 분석한다.
5.  **키워드 분기:**
    * `begin` / `end`: `foundMain` 플래그가 1일 때 (main 함수 내부일 때)만 `type 4` 또는 `type 5` 노드를 `Push`한다.
    * `function`: `strtok`로 함수 이름을 파싱한다.
        * `type 2` 노드를 생성하고 함수 이름(exp\_data)과 정의된 라인 번호(line)를 `STACK`에 `Push`한다.
        * 함수 이름이 `main`이면 `foundMain = 1` 플래그를 세운다.
        * `foundMain`이 1이고 함수가 `main`이 아니면(즉, `main` 이후에 정의된 다른 함수를 파싱 중이면), 파라미터 변수(예: `f(int a)`)를 `type 1` 노드로 `STACK`에 `Push`한다. 이때 값은 `CalingFunctionArgVal` 에서 가져온다.
    * `(` (수식): [6-B. 수식 계산](#b-수식-계산--키워드-처리) 참조.
    * `end` (핵심 로직):
        * `GetLastFunctionCall(STACK)`을 호출하여, 현재 `end`가 함수의 `end` 인지 (반환값 > 0), `main`의 `end`인지(반환값 == 0) 확인한다.
        * **Main의 `end` (반환값 0):** `LastExpReturn` (최종 계산 결과)을 `printf`로 출력하고 프로그램이 사실상 종료된다.
        * **함수의 `end` (반환값 > 0):**
            1.  함수의 최종 계산 값(`LastExpReturn`)을 `LastFunctionReturn`에 백업한다.
            2.  `fclose(filePtr)`와 `fopen(argv[1], "r")`을 사용해 파일 포인터를 **처음으로 되돌린다.**
            3.  `fgets` 루프를 통해 함수를 호출했던 라인(`sline`) 직전까지 이동한다.
            4.  `STACK`에서 `Pop`을 반복하며 `Type 3` (함수 호출) 노드를 찾아 제거한다. (호출 스택 정리)
6.  **종료:** 루프가 끝나면(파일 끝) `fclose(filePtr)`로 파일을 닫고 `FreeAll(STACK)`로 모든 메모리를 해제한 뒤 종료한다.

<br>

## 6. 핵심 기능 분석 (Line by Line)

### A. 스택 관리 (Push / Pop)

* **`Push`**, **`PushOp`**, **`PushPostfix`** : `malloc` 으로 새 노드를 할당하고, 값을 채운 뒤, 스택의 `top`을 이 새 노드로 교체한다. (연결 리스트의 헤드에 삽입)
* **`Pop`**, **`PopOp`**, **`PopPostfix`** : `top` 노드의 데이터를 임시 변수에 저장하고, `top`을 `top->next`로 이동시킨 뒤, 기존 `top` 노드를 `free` 한다.

### B. 수식 계산 (`(` 키워드 처리)

수식 라인(예: `((b+c)/a)`)이 감지되면, 두 단계로 처리된다.

> **[그림 제안 2: 수식 계산 과정]**
> 이 부분에 `input1.spl`의 `((b+c)/a)`가 `(6+2)/4`로 변환되고, 최종 결과 `2`가 나오는 과정을 2단계로 나누어 그립니다.
>
> 1.  **Infix -> Postfix 변환:** `lineyedek`(입력), `postfix` 배열(출력), `MathStack`(임시)의 상태 변화를 표로 보여줍니다. (결과: `62+4/`)
> 2.  **Postfix 계산:** `postfix` 배열을 읽으면서 `CalcStack`에 값이 PUSH/POP되고 연산되는 과정을 보여줍니다. (결과: `2`)
>
> ``
> ``

**1단계: 중위(Infix) -> 후위(Postfix) 변환**

* `lineyedek` (원본라인)을 한 글자(`char`)씩 순회한다.
* **숫자/변수:** `postfix` 배열에 바로 추가한다.
    * 변수인 경우 (`isalpha`): `GetVal`을 호출한다.
* **연산자 (+,-,\*,\/):**
    * `MathStack`이 비어있으면 `PushOp` .
    * `MathStack`에 연산자가 있으면, `Priotry` 함수로 우선순위를 비교한다.
    * 새 연산자의 우선순위가 스택 `top`의 연산자보다 낮거나 같으면, 스택 `top`을 `PopOp`하여 `postfix` 배열에 추가하고, 새 연산자를 `PushOp` 한다.
    * 새 연산자의 우선순위가 높으면, 그냥 `PushOp`.
* **`)` (닫는 괄호):** `MathStack`에서 `PopOp`을 하여 `postfix` 배열에 추가한다.
* **최종:** `MathStack`에 남은 모든 연산자를 `PopOp`하여 `postfix` 배열에 추가한다.

**2단계: 후위(Postfix) 수식 계산**

* 변환된 `postfix` 배열을 한 글자씩 순회한다.
* **숫자:** `postfix[i] - '0'` 을 통해 정수로 변환 후 `CalcStack` 에 `PushPostfix` 한다.
* **연산자:**
    * `val1 = PopPostfix(CalcStack)` (두 번째 피연산자)
    * `val2 = PopPostfix(CalcStack)` (첫 번째 피연산자)
    * `switch` 문을 통해 `val2 + val1` (혹은 -, *, /)연산을 수행한다.
    * 결과(`resultVal`)를 다시 `CalcStack`에 `PushPostfix`한다.
* **최종:** `postfix` 배열 순화가 끝나면 `CalcStack->top->val`에 최종 계산 결과가 남는다. 이 값을 `LastExpReturn`에 저장한다.

### C. 변수/함수 값 조회 (GetVal)

`GetVal(char exp_name, int* line, Stack* stck)` 함수는 이 인터프리터의 '심볼 테이블' 역할을 한다.

> **[그림 제안 3: GetVal과 변수 스코프]**
> `f` 함수 내부에서 `GetVal('c')`를 호출하는 시점의 `STACK` 상태를 그립니다.
>
> (예시 `STACK` 그림)
> `Top -> [type: 1, data: 'c', val: 2] (f의 c)`
> `... -> [type: 1, data: 'b', val: 6] (f의 b)`
> `... -> [type: 1, data: 'a', val: 4] (f의 a)`
> `... -> [type: 3, line: 14] (f 호출 정보)`
> `... -> [type: 1, data: 'c', val: 4] (main의 c)`
> `...`
>
> 이 그림을 통해 `GetVal`이 `top`에서부터 검색하므로 `f`의 `c=2`가 `main`의 `c=4`보다 먼저 반환됨을 보여줍니다. (스코프 구현)
>
> ``

1.  `STACK` 의 `top` (가장 최신 데이터)부터 `head` 포인터로 순회한다.
2.  `head->exp_data == exp_name` (찾는 이름과 일치하는지) 확인한다.
3.  **일치하는 노드 발견 시:**
    * `head->type == 1` (변수): `head->val` (변수 값)을 반환한다. (가장 나중에 선언된 변수, 즉 현재 스코프의 변수가 먼저 검색됨)
    * `head->type == 2` (함수): 포인터로 받는 `line` 변수에 `head->line` (함수 정의 라인)을 저장하고, -1을 반환한다.
4.  못 찾으면 -999를 반환한다.

### D. 함수 호출 및 복귀 (Call & Return)

이 인터프리터는 재귀적인 C 함수 호출이 아닌, **파일 포인터를 되감는 (rewind)** 방식으로 함수 호출을 흉내 낸다.

> **[그림 제안 4: 함수 호출/복귀 흐름도]**
> 이 인터프리터에서 가장 복잡하고 핵심적인 로직입니다. 파일 포인터(`filePtr`)와 `STACK`의 상태 변화를 2단계로 나누어 그립니다.
>
> 1.  **함수 호출 (Call) 시점 (14라인):**
>     * `filePtr`가 14라인을 가리킴.
>     * `GetVal('f')` 호출 -> `STACK`에 `[type: 3, line: 14]` (복귀 주소) PUSH.
>     * `fclose()` -> `fopen()`.
>     * `filePtr`가 1라인으로 이동함.
>
> 2.  **함수 복귀 (Return) 시점 (6라인):**
>     * `filePtr`가 6라인을 가리킴.
>     * `GetLastFunctionCall()` 호출 -> `STACK`에서 `[type: 3, line: 14]` 찾음 (복귀 주소 14 획득).
>     * `f`의 지역 변수/파라미터를 `STACK`에서 `Pop` (스택 정리).
>     * `fclose()` -> `fopen()`.
>     * `filePtr`가 14라인으로 이동함.
>
> ``
> ``

**호출 (Call) 과정 (`(` 수식 처리 중 `GetVal`이 -1 반환 시):**

1.  수식 `f(c)`에서 `c` 의 값을 `GetVal`로 찾아 `CalingFunctionArgVal`에 저장한다. (예: 4)
2.  `STACK`에 `Type 3` (함수 호출) 노드를 `Push` 한다. 이 노드에는 현재 라인 번호 (`curLine`)가 기록된다. (복귀할 위치)
3.  `GetVal`이 찾아준 `codeline` (함수 `f`가 정의된 라인)으로 이동하기 위해, `fclose` -> `fopen` -> `fgets` 루프를 실행한다.
4.  `WillBreak = 1` 플래그를 세워, 수식 계산을 중단하고, 메인 루프로 돌아간다.

**복귀 (Return) 과정 (`end` 키워드 처리 중):**

1.  함수 `f`의 `end` 라인에 도달한다.
2.  `GetLastFunctionCall(STACK)`는 `type 3` 노드를 찾아 `STACK`에 저장된 복귀 위치를 반환한다.
3.  함수의 최종값 `LastExpReturn`을 `LastFunctionReturn`에 저장한다.
4.  다시 `fclose` -> `fopen` -> `fgets` 루프를 실행하여 복귀 위치(14라인) 직전까지 이동한다.
5.  `STACK`에서 `type 3` 노드가 나올 때까지 `Pop`을 반복한다.(함수 `f`에서 사용된 지역 변수 `b`, `c`, 파라미터 `a`, 함수 정의 `f`를 모두 스택에서 제거)
6.  `willBreak = 0` 이므로, 14라인의 수식 `((6 + f(c) ) / b)`를 다시 계산(re-evaluating)한다.
7.  수식 계산 중 `f`를 만나 `GetVal`을 호출하면, 이번에는 `LastFunctionReturn != -999` (즉, 2)이므로, 이 값은 `f(c)`의 결과로 사용하여 수식을 최종 계산한다.
8.  `LastFunctionReturn = -999`로 리셋한다.

<br>

## 7. Input1.spl 실행 추적 (Step-by-Step)

**`input1.spl` 코드**
```spl
function f(int a)  (Line 1)
begin
    int b = 6;
    int c = 2;
    ((b+c)/a);     (Line 5)
end
function main()    (Line 8)
begin
    int a = 1;
    int b = 2;
    int c = 4;
    ((6 + f(c) ) / b); (Line 14)
end
```
