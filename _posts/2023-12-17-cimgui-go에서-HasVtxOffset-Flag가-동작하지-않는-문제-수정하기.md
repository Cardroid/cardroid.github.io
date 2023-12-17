---
title: cimgui-go에서 HasVtxOffset Flag가 동작하지 않는 문제 수정하기
author: cardroid
date: 2023-12-17 12:00:00 +0900
categories: [일기, Fix]
tags: [golang, imgui, bug]
---

> 현재 main 브랜치 cimgui-go가 Windows에서 빌드가 되지 않는 문제로 로컬에서 따로 작업했습니다.
{: .prompt-info }

나는 최근 음성합성 인공지능에 필요한 데이터를 준비하는 중, 기존 라벨링 툴들을 사용하다 답답해서 새로 개발 중인 툴이 있다.
이름은 `COCOA`인데, 공부할 겸 go언어를 사용하여 개발하고 있다.   

오디오는 백엔드로 `midiaudio`, GUI는 심플하게 [Imgui](https://github.com/ocornut/imgui)를 사용 중인데, 한 가지 문제가 생겨서 시간을 많이 허비했다...   
이 과정을 되풀이하지 않기 위해 기록을 남긴다.   

---

## Imgui 소개
> Imgui는 다양한 플랫폼과 렌더러를 추상화하여 간단하게 GUI를 구현할 수 있는 크로스 플랫폼 즉시 모드 UI 라이브러리다.   

Imgui에서는 플랫폼과 렌더러를 백엔드라고 부르는 것 같은데, 구체적이게 하는 일은 플랫폼 백엔드가 각종 IO를 담당하고 렌더러 백엔드가 화면을 그리는 작업을 담당한다.   
자세한 정보는 [Imgui-Backend](https://github.com/ocornut/imgui/blob/master/docs/BACKENDS.md)에서 확인이 가능하다.   
나는 예전에 Python으로 사용해 본 적(사용하다 느려서 버렸다)이 있어서 바로 결정했다.   

이 라이브러리는 기본 C++ 언어로 지원되기 때문에 래핑을 하여 사용하여야 하는데, go 언어 바인딩을 찾아보았지만 오래된 바인딩밖에 없고 내가 사용하려는 Implot 함수가 없는 버전이라, C언어 자동 바인딩을 래핑하는 [cimgui-go](https://github.com/AllenDang/cimgui-go)를 사용했다.   

즉 `C++ Imgui -> (Auto Wrapping) C Imgui -> (Auto Wrapping) Go Imgui` 이렇게 된다.   
(cgo가 overhead가 생각보다 있다고 해서 좋아 보이진 않은 것 같다)

## 문제점
Imgui는 ImDrawList를 사용하여 화면에 그릴 UI 정점들을 관리하고 렌더러에 전달해 주는데, 여기서 ImDrawIdx가 16-bit `unsigned short` 형이라 64,000개 이상의 정점을 그릴 수 없는 문제가 있었다.   

사실 [이미 해결된 문제](https://github.com/ocornut/imgui/issues/2591)인데, cimgui-go에서는 `ImGuiBackendFlags_RendererHasVtxOffset` Flag를 설정해도 동작하지 않는 문제가 발생했다.   

## 이게 왜 문제일까
내가 개발하고 있는 COCOA는 음성 라벨링을 하기 위해 오디오 Waveform을 그래프로 그려야 하는데, 이 데이터가 매우 클 수 있어서 문제가 되었다.   

성능 문제로 LTTB 다운샘플링도 적용할 예정이지만, 모든 UI 구성요소를 포함하여 64,000개의 정점으로는 많이 부족했고, 따라서 해결을 해보려고 시도했다.   

## 접근
일단 디버깅을 시도하려고 했지만, cgo가 윈도우의 심볼을 지원하지 않는 문제인지, 아니면 디버그 심볼 없이 빌드 되는 옵션을 적용했는지, 잘되지 않았다.

그래서 cimgui-go의 해석을 하려고 했는데, cimgui의 래퍼 프로젝트라... (cimgui도 찾아보고 Lua를 사용하는 코드 생성기라 이것도 만져보고, 다른 문제도 여려 개가 발생해서...) cgo 문서와 github CI/CD 구성을 보고 어떻게든 테스트 코드를 삽입할 수 있는 방법을 찾아내었다.   

기존 imgui + 자동 생성된 cimgui/go 소스코드 + 타입/함수 json 사전 및 템플릿의 콤보로 조금 헤매었지만, ImDrawData가 glut_opengl3에서 렌더링 되는 함수를 찾을 수 있었다.

여기서 전처리문으로 OpenGL 3.2 이상에서만 VtxOffset이 동작하도록 되어있어서 왜 동작하지 않는지 의아했는데, OpenGL이 3.2부터 프로파일이 생겨서, 현재 버전은 3.2 이상이지만 이전 버전의 호환 프로필로 동작할 수 있다는 문서를 보고 버전을 3.2 이상으로 고정했다.   

그랬더니 다른 수정이 필요 없이 깔끔하게 동작했다...

## END
덕분에 WSL도 돌리고 Luajit도 사용해 보고 삽질을 많이 했지만 cgo 관련해서도 배울 수 있었고, 여러모로 얻은 것도 많은 것 같다.

이걸로 이슈도 열었는데.. 😅