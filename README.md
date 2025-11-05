# Basic-Interpreter-basic_interpreter.c-
OS 오전반  / 도현명

## 1. 과제 목적
본 문서는 basic_interpreter.c로 작성된 C 기반 인터프리터의 소스 코드를 분석하는 것을 목적으로 한다. 프로그램의 전체 구조를 이해하고, 입력 파일(.SPL)이 어떤 절차를 거쳐 최종 출력을 생성하는지 코드 레벨에서 추적한다.
이 문서는 핵심 자료구조, 주 실행 흐름, 함수 호출 관계를 다이어그램과 상세한 설명을 통해 기술한다.

## 2. 핵심 자료구조 
이 인터프리터는 C의 STRUCT와 포인터를 이용해 3가지 핵심 스택(Stack)을 구현하여 프로그램을 실행한다.

## A.메인 스택(STACK)
- 구조체: struct node / Stack
- 용도: 프로그램의 모든 '상태'를 저장하는 가장 중요한 스택.
- 저장 내용 (node.type):
  - type 1:변수 (Variable).(예: int a=1)
  - type 2:함수 정의 (Function Defintion).(예:function f)
  - type 3:함수 호출 (Fuction Call).(예:f(c))
  - type 4:블록 시작 (Begin).
  - type 5:블록 종료 (End)


