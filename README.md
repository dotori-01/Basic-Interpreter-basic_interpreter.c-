# Basic-Interpreter-basic_interpreter.c-
OS 오전반  / 도현명

## 1. 과제 목적
본 문서는 `basic_interpreter.c`로 작성된 C 기반 인터프리터의 소스 코드를 분석하는 것을 목적으로 한다. 프로그램의 전체 구조를 이해하고, 입력 파일(`.SPL`)이 어떤 절차를 거쳐 최종 출력을 생성하는지 코드 레벨에서 추적한다.
이 문서는 핵심 자료구조, 주 실행 흐름, 함수 호출 관계를 다이어그램과 상세한 설명을 통해 기술한다.

## 2. 핵심 자료구조 
이 인터프리터는 C의 `struct`와 포인터를 이용해 3가지 핵심 스택(Stack)을 구현하여 프로그램을 실행한다.

### A.메인 스택 (STACK)
- 구조체: `struct node` / `Stack`
- 용도: 프로그램의 모든 '상태'를 저장하는 가장 중요한 스택.
- 저장 내용 (node.type):
  - `type 1`:변수 (Variable).(예: int a=1)
  - `type 2`:함수 정의 (Function Defintion).(예:function f)
  - `type 3`:함수 호출 (Fuction Call).(예:f(c))
  - `type 4`:블록 시작 (Begin).
  - `type 5`:블록 종료 (End)
- 특징:변수의 스코프(Scope)와 함수 호출/복귀를 이 스택을 통해 관리한다.
`/* 1 var, 2 function, 3 function call, 4 begin, 5 end */
struct node {
    int type; 
    char exp_data;
    int val;
    int line;
    struct node* next;
};`

### B.연산자 스택 (MathStack)
- 구조체: `struct opnode` / `onstack`
- 용도:수식 계산 시, 중위 표기법(Infix)을 후위 표기법(Postfix)으로 변환하기 위해 연산자(+,-,*,/)를 임시 저장한다.`
  struct opnode { 
    char op; 
    struct opnode* next; 
};`

### C.피연산자 스택 (CalcStack)
- 구조체: `struct postfixnode` / `PostfixStack`
- 용도:후위 표기법(Postfix)으로 변환된 수식을 실제 계산하기 위해 피연산자(숫자)를 저장한다.
 ` struct postfixnode { 
    int val; 
    struct postfixnode* next; 
};`

## 3.프로그램 전체 실행 흐름 (main 함수)
`main`함수는 인터프리터의 전체 실행을 주관한다.
1. 초기화: `MathStack`, `CalcStack`, `STACK` 3개의 스택을 위한 메모리를 할당한다.
2. 인자 확인: `ARGC != 2`인지 확인하여, 프로그램 실행 시 `.SPl` 파일이 정확히 1개 주어졌는지 검사한다.
3. 파일 열기: `fopen(argv[1], "r")`로 입력 `.spl`파일을 연다.
4. 메인 파싱 루프: while` (fgets(line, 4096, filePtr))`
 - 파일을 한 줄씩 읽어 `line` 버퍼에 저장한다.
 - `curLine`을 증가시켜 현재 라인 번호를 추적한다.
 - `rstrip()`으로 줄 끝의 공백/개행 문자를 제거한다.
 - `my_stricmp()`(대소문자 무시 문자열 비교)를 사용해 키워드를 분석한다.
5. 키워드 분기:
  - `begin`/ `end`:`foundMain` 플래그가 1일 때 (main 함수 내부일 때)만 `type 4` 또는 `type 5` 노드를 `Push`한다.
  - `funtion`: `strtok`로 함수 이름을 파싱한다.
     - `type 2` 노드를 생성하고 함수 이름(exp_data)과 정의된 라인 번호(line)를 `STACK`에 `Push`한다.
     - 함수 이름이 `main`이면 `foundMain = 1` 플래그를 세운다.
     - `foundMain이` 1이고 함수가 `main`이 아니면(즉, `main` 이후에 정의된 다른 함수를 파싱 중이면), 파라미터 변수(예: `f(int a))`를 `type 1` 노드로 `STACK`에 `Push`한다. 이때 값은 `CalingFunctionArgVal` 에서 가져온다.
     - `(` (수식):4. 핵심 기능 분석- B.수식 계산 참조.
     - `end` (핵심 로직):
        - `GetLastFunctionCall(STACK)`을 호출하여, 현재 `end`가 함수의 `end` 인지 (반환값 >0), `main`의 `end`인지(반환값 == 0) 확인한다.
       - Main의 `end` (반환값 0): `LastExpReturn` (최종 계산 결과)을 `printf`로 출력하고 프로그램이 사실상 종료된다.
       - 함수의 `end` (반환값 >0):
         1. 함수의 최종 계산 값(`LastExpReturn`)을 `LastFunctionReturn`에 백업한다.
         2. `fclose(filePtr)`와 `fopen(argv[1], "r")`을 사용해 파일 포인터를 처음으로 되돌린다.
         3. `fgets` 루프를 통해 함수를 호출했던 라인(`sline`) 직전까지 이동한다.
         4. `STACK`에서 `Pop`을 반복하며 `Type 3` (함수 호출) 노드를 찾아 제거한다. (호출 스택 정리)
         5.  종료: 루프가 끝나면(파일 끝) `fclose(filePtr)`로 파일을 닫고 `FreeAll(STACK)`로 모든 메모리를 해제한 뒤 종료한다.

## 4. 핵심 기능 분석 (Line by Line)

### A. 스택 관리 (Push / Pop)
- `Push`, `PushOp`, `PushPostfix` : `malloc` 으로 새 노드를 할당하고, 값을 채운 뒤, 스택의 `top`을 이 새 노드로 교체한다.  (연결 리스트의 헤드에 삽입)
- `Pop`, `PopOp`, `PopPostfix` : `top` 노드의 데이터를 임시 변수에 저장하고, `top`을 `top->next`로 이동시킨 뒤, 기존 `top` 노드를 `free` 한다.
        
### B.수식 계산(`(`키워드 처리)
수식 라인(예:`((b+c)/a`)이 감지되면, 두 단계로 처리된다.

1단계: 중위(Infix) -> 후위(Postfix) 변환
- `lineyedek` (원본라인)을 한 글자(`char`)씩 순화한다.
- 숫자/변수: `postfix` 배열에 바로 추가한다.
   - 변수인 경우 (`isalpha`): `GetVal`을 호출한다.
- 연산자 (+,-,*,/):
  - MathStack이 비어있으면 PushOp .
  - MathStack에 연산자가 있으면, Priotry 함수로 우선순위로 비교한다.
  - 새 연산자의 우선순위가 스택 top의 연산자보다 낮거나 같으면, 스택 top을 PopOp하여 postfix 배열에 추가하고, 새 연산자를 PushOp 한다. (예: a*b+ 상태에서 c가 들어오면 *를 pop함)
  - 새 연산자의 우선순위가 높으면, 그냥 PushOp.
- `)` (닫는 괄호): `MathStack`에서 `PopOp`을 하여 `postfix` 배열에 추가한다.(`(`는 스택에 넣지 않으므로 `(`가 나올 때까지 `pop`할 필요가 없는 단순한 구현)
- 최종: `MathStack`에 남은 모든 연산자를 `PopOp`하여 `postfix` 배열에 추가한다.

2단계:후위(Postfix) 수식 계산
 - 변환된 `postfix` 배열을 한 글자씩 순화한다.
 - 숫자: `postfix[i] - '0'` 을 통해 정수로 변환 후 `CalcStack` 에 `PushPostfix` 한다.
 - 연산자:
   - `val1 = PopPostfix(CalcStack)` (두 번째 피연산자)
   - `val2 = PopPostfix(CalcStack)` (첫 번째 피연산자)
   - `switch` 문을 통해 `val2 + val1` (혹은 -,*,/)연산을 수행한다.
   - 결과(`resultVal`)를 다시 `CalcStack`에 `PushPostfix`한다.
- 최종: `postfix` 배열 순화가 끝나면 `CalcStack->top>val`에 최종 계산 결과가 남는다. 이 값을 `LastExpReturn`에 저장한다.

C. 변수/함수 값 조회 (GetVal)
GetVal(char exp_name, int* line, Stack* stck) 함수는 이 인터프리터의 '심볼 테이블' 역할을 한다.
1. STACK 의 top (가장 최신 데이터)부터 head 포인터로 순화한다.
2. head->exp_data == exp_name (찾는 이름과 일치하는지) 확인한다.
3. 일치하는 노드 발견 시:
  - head->type == 1 (변수):head->val (변수 값)을 변환한다. (가장 나중에 선언된 변수, 즉 현재 스코프의 변수가 먼저 검색됨)
  - head->type == 2 (함수): 포인터로 받는 line 변수에 head->line (함수 정의 라인)을 저장하고, -1을 반환한다.
4. 못 찾으면 -999를 반환한다.

D. 함수 호출 및 복귀 (Call & Return)
이 인터프리터는 재귀적인 C 함수 호출이 아닌, 파일 포인터를 되감는 (rewind) 방식으로 함수 호출을 흉내 낸다.

호출 (Call) 과정 ((수식 처리 중  GetVal이 -1 반환 시):
1. 수식 f(c)에서 c 의 값을 GetVal로 찾아 CalingFuctionArgVal에 저장한다. (예:4)
2. `STACK`에 `Type 3` (함수 호출) 노드를 Push 한다. 이 노드에는 현재 라인 번호 (`curLine`)가 기록된다. (복귀할 위치)
3. `GetVal`이 찾아준 codeline (함수 f가 정의된 라인)으로 이동하기 위해, fclose -> `fopen` -> `fgets` 루프를 실행한다.
4. `WillBreak = 1` 플래그를 세워, 수식 계산을 중단하고, 메인 루프로 돌아간다.
복귀 (Return)과정 (end 키워드 처리 중):
1. 함수 f의 end 라인에 도달한다.
2. GetLastFunctionCall(Stack)는 type 3 노드를 찾아 STACK에 저장된 복귀 위치를 반환한다.
3. 함수의 최종값 LastExpReturn을 LastFunctionReturn에 저장한다.
4. 다시 fclose -> fopen -> fgets 루프를 실행하여 복귀 위치(14라인) 직전까지 이동한다.
5. STACK에서 type 3 노드가 나올 때까지 Pop을 반복한다.(함수 f에서 사용된 지역 변수 b, c, 파라미터 a, 함수 정의 f를 모두 스택에서 제거)
6. willBreak = 0 이므로, 14라인의 수식 ((6 + f(c) / b)를 다시 계산(re-evaluating)한다.
7. 수익 계산 중 f를 만나 GetVal을 호출하면, 이번에는 LastFunctionReturn != -999 (즉,2)이므로, 이 값은 f(c)의 결과로 사용하여 수식을 최종 계산한다.
8.  LastFunctionReturn = -999로 리셋한다.

## 5. input1.spl 실행 추적 (Step-by-Step)
input.spl 코드
`function f(int a)  (Line 1)
begin
   int b = 6;
   int c = 2;
   ((b+c)/a);      (Line 5)
end
function main()   (Line 8)
begin
   int a = 1;
   int b = 2;
   int c = 4;
   ((6 + f(c) ) / b); (Line 14)
end`
outputs.txt: input1.spl -> 4


## 6. 결론
이 Basic Interpreter는 C언어의 파일 처리, 동적 메모리 할당, 포인터를 활용한 스택 자료구조를 통해 구현되었다.

특히 함수 호출을 C의 재귀 호출(Call Stack)에 의존하지 않고, STACK 자료구조와 파일 포인터(fseek/rewind 대신 fclose/fopen)를 조합하여 직접 '실행 문맥'을 관리하는 방식이 인상적이다. 이는 운영체제의 컨텍스트 스위칭(Context Switching)이나 CPU의 프로그램 카운터(PC) 동작 원리를 단순화하여 보여주는 훌륭한 예시이다.

input3.spl과 input4.spl에 if 키워드가 있지만, basic_interpreter.c 코드에는 if를 처리하는 로직이 전혀 없다. strtok로 첫 단어를 분리할 때 if를 무시하고 다음 라인으로 넘어가기 때문에 if 조건과 관계없이 다음 줄의 수식을 실행하거나 무시하게 된다. (예: input4.spl은 if ( c > b )가 참이므로 (1+g(x))가 실행되어 5가 됨). 이는 인터프리터의 불완전한 파싱 로직을 보여준다.




