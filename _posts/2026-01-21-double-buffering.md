---

layout: post

title:  "[C언어로 게임만들기] 02. 눈 아픈 깜빡임은 이제 그만! 더블 버퍼링(Double Buffering) 마법"

date:   2026-01-21 16:30:00 +0900

categories: [C언어로 게임만들기]

tags: [C언어로 게임만들기, C언어, 게임개발, 더블버퍼링, 윈도우API]

---



## 제발 `system("cls")` 좀 그만 써!



C언어로 게임을 처음 만들 때 가장 많이 하는 실수가 있습니다. 바로 화면을 지울 때 `system("cls")` 명령어를 사용하는 것입니다.



이걸 매 프레임마다(`while` 문 안에서) 사용하면 어떻게 될까요? 화면이 \*\*번쩍번쩍거리며 눈뽕(?)\*\*을 선사하게 됩니다. 화면을 지우고 다시 그리는 과정이 우리 눈에 다 보이기 때문이죠.



오늘은 프로들의 기술, \*\*더블 버퍼링(Double Buffering)\*\*을 구현해서 영화처럼 부드러운 화면을 만들어보겠습니다.



### 1. 더블 버퍼링이란?



원리는 간단합니다. \*\*화면(Buffer)을 두 개\*\* 만드는 겁니다.



1.  \*\*화면 A:\*\* 사용자에게 보여주는 화면

2.  \*\*화면 B:\*\* 뒤에서 몰래 다음 장면을 그리는 화면



사용자가 화면 A를 보고 있을 때, 우리는 화면 B에 열심히 그림을 그립니다. 다 그리면 \*\*순식간에 A와 B를 바꿔치기(Flip)\*\* 합니다. 이렇게 하면 그리는 과정이 보이지 않아 깜빡임이 사라집니다.



---



\### 2. 소스 코드 구현



이 기능을 별도의 파일로 깔끔하게 분리하겠습니다. `ScreenBuffer.h`와 `ScreenBuffer.cpp` 파일을 프로젝트에 추가해주세요.



\#### \[헤더 파일] ScreenBuffer.h

다른 파일에서 이 기능을 가져다 쓸 수 있도록 "메뉴판"을 만듭니다.



```cpp

#pragma once

#include <tchar.h>



// 화면 크기 상수 정의 (80칸 x 25줄)

const int SCREEN\_WIDTH = 80;

const int SCREEN\_HEIGHT = 25;



// 함수 선언부

void ScreenInit();      // 화면 초기화 (버퍼 생성)

void ScreenFlipping();  // 화면 교체 (보여주기)

void ScreenClear();     // 화면 지우기

void ScreenRelease();   // 메모리 해제

void ScreenPrint(int x, int y, const wchar\_t\* string); // 글자 찍기

void SetColor(unsigned short color); // 색깔 바꾸기

```

---



```cpp

#include "ScreenBuffer.h"

#include <windows.h>



static int g_nScreenIndex;

static HANDLE g_hScreen\[2]; // 화면 2개 (Front, Back)

static int g_wColor = 0x0007; // 기본 색상: 흰색



void ScreenInit()

{

  // 1. 콘솔 창 크기 명령 (1차 시도)

   system("mode con: cols=80 lines=25");



   CONSOLE_CURSOR\_INFO cci;

   COORD size = { SCREEN_WIDTH, SCREEN_HEIGHT };

   SMALL_RECT rect = { 0, 0, SCREEN_WIDTH - 1, SCREEN_HEIGHT - 1 };



   // 2. 버퍼 2개 생성 (핵심!)

   g_hScreen[0] = CreateConsoleScreenBuffer(GENERIC_READ | GENERIC_WRITE, 0, NULL, CONSOLE_TEXTMODE_BUFFER, NULL);
   g_hScreen[1] = CreateConsoleScreenBuffer(GENERIC_READ | GENERIC_WRITE, 0, NULL, CONSOLE_TEXTMODE_BUFFER, NULL);



   // [핵심] 3. 버퍼 크기와 윈도우 크기를 강제로 맞춤 (스크롤바 제거 \& 잔상 해결)

   for (int i = 0; i < 2; i++)
   {
       SetConsoleScreenBufferSize(g\_hScreen\[i], size); // 버퍼 크기 고정

       SetConsoleWindowInfo(g\_hScreen\[i], TRUE, \&rect); // 창 크기 고정



       // 커서(깜빡이는 밑줄) 숨기기

       cci.dwSize = 1;

       cci.bVisible = FALSE;

       SetConsoleCursorInfo(g\_hScreen\[i], \&cci);

  }

}



void ScreenFlipping()

{

   Sleep(10); // 너무 빠르면 안 보이니까 살짝 대기

   SetConsoleActiveScreenBuffer(g_hScreen[g_nScreenIndex]); // 보여주는 화면 교체

   g_nScreenIndex = !g_nScreenIndex; // 0 -> 1, 1 -> 0 스위칭

}



void ScreenClear()

{

   COORD Coor = { 0, 0 };

   DWORD dw;

   // 화면 전체를 공백(' ')으로 채워서 지움

   FillConsoleOutputCharacter(g\_hScreen\[g\_nScreenIndex], ' ', 80 \* 25, Coor, \&dw);

   // 색상 정보도 초기화

   FillConsoleOutputAttribute(g\_hScreen\[g\_nScreenIndex], 0x0007, 80 \* 25, Coor, \&dw);

}



void ScreenRelease()

{

  CloseHandle(g_hScreen\[0]);

  CloseHandle(g_hScreen\[1]);

}



void ScreenPrint(int x, int y, const wchar_t* string)

{

   DWORD dw;

   COORD CursorPosition = { x, y };

   int len = (DWORD)wcslen(string); // 문자열 길이 계산

 // 커서 위치를 잡고 글자를 씀

   SetConsoleCursorPosition(g_hScreen[g_nScreenIndex], CursorPosition);

   WriteConsoleOutputCharacterW(g_hScreen[g_nScreenIndex], string, len, CursorPosition, &dw);

  FillConsoleOutputAttribute(g_hScreen[g_nScreenIndex], g_wColor, len, CursorPosition, &dw);

}



void SetColor(unsigned short color)

{
   g_wColor = color;

}

```



### 2. 게임 루프에 적용하기 (사용법)

지난 시간에 만든 \*\*게임 루프(main.c)\*\*에 이 기능을 연결해 봅시다. 



| 게임 루프 단계 | 호출할 Screen 함수 | 비고 |

| :--- | :--- | :--- |

| \*\*Awake\*\* (초기화) | `ScreenInit()` | 창을 만들고 커서를 숨김 |

| \*\*PreRender\*\* (지우기) | `ScreenClear()` | 뒷면 도화지를 깨끗이 지움 |

| \*\*Render\*\* (그리기) | `ScreenPrint(...)` | 뒷면 도화지에 그림을 그림 |

| \*\*PostRender\*\* (보여주기) | `ScreenFlipping()` | 다 그린 뒷면을 앞으로 짠! 하고 보여줌 |

| \*\*Release\*\* (해제) | `ScreenRelease()` | 메모리 정리 |



---



### 마무리: 이제 진짜 게임 화면 같습니다!



오늘 우리는 게임 개발의 난제 중 하나인 \*\*'화면 깜빡임(Flickering)'\*\*을 더블 버퍼링 기법으로 완벽하게 해결했습니다.



\* \*\*Before:\*\* `system("cls")`로 인해 눈이 아프고 뚝뚝 끊기는 화면

\* \*\*After:\*\* 영화 필름처럼 부드럽게 전환되는 전문가 수준의 콘솔 화면



이 `ScreenBuffer` 코드는 앞으로 우리가 만들 \*\*모든 콘솔 게임의 도화지\*\*가 될 것입니다. 유니티가 내부적으로 어떻게 화면을 그리는지 직접 구현해 보니 감회가 새롭지 않나요?



하지만 아직 화면에 점 하나만 찍혀있을 뿐, 움직이지는 않습니다. 다음 시간에는 이 멈춰 있는 점을 내 마음대로 조종할 수 있도록 \*\*키보드 입력 처리(Input System)\*\*를 구현해 보겠습니다.



이제 진짜 캐릭터가 살아 움직이게 될 테니 기대해 주세요!








