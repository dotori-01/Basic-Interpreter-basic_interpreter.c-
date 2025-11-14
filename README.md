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

### B. 연산자 스택 (MathStack)

* **구조체:** `struct opnode` / `OpStack` (코드에는 `opstack`으로 되어 있으나 편의상 `MathStack`으로 칭함)
* **용도:** 수식 계산 시, 중위 표기법(Infix)을 후위 표기법(Postfix)으로 변환하기 위해 연산자(+, -, *, /)를 임시 저장한다.

```c
struct opnode { 
    char op; 
    struct opnode* next; 
};
