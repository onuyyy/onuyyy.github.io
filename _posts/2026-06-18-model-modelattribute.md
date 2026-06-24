---
title: "Model & @ModelAttribute 정리"
date: 2026-06-18 11:00:00 +0900
categories: [Spring]
tags: [Spring, MVC, Model]
---

컨트롤러에서 **요청 값을 받고**, 그 값을 **뷰(JSP)로 넘기는** 방법 정리.

> 핵심 두 동작: **① 받기**(요청 파라미터 가져오기) + **② 담기**(모델에 넣어 뷰로 전달).
> 어떤 어노테이션이 둘 중 무엇을 해주는지가 전부다.

---

## 1. Model 이란

- 컨트롤러 → 뷰로 데이터를 넘기는 **상자**. 여기에 담은 값은 JSP에서 `${이름}`으로 꺼낸다.
- **메서드 실행 시 스프링이 항상 내부에 하나 만들어 둔다.** `Model model` 파라미터는 그 상자에 접근하는 손잡이일 뿐, 안 받는다고 상자가 사라지는 게 아니다.
- 그래서 `Model`을 파라미터로 안 받아도, 자동으로 담기는 값(아래 `@ModelAttribute`)은 뷰로 잘 넘어간다. **직접 추가로 담고 싶을 때만** `Model`을 받으면 된다.

```java
public void ex(Model model) {
    model.addAttribute("now", new Date());  // 수동으로 담기 → 뷰에서 ${now}
}
```

---

## 2. "받기"와 "담기"는 별개 동작

| 어노테이션/객체 | ① 받기(요청값) | ② 담기(뷰 전달) |
|------|:---:|:---:|
| `@ModelAttribute` | ✅ | ✅ (자동) |
| `@RequestParam` | ✅ | ❌ |
| `Model.addAttribute` | ❌ | ✅ (수동) |

- `@ModelAttribute` = `@RequestParam`(받기) + `model.addAttribute`(담기)를 **한 번에** 하는 것.
- `model.addAttribute`는 "담기"만 한다 → 담을 값을 **어디선가 이미 받아와야** 쓸 수 있다.

---

## 3. @ModelAttribute — 받기 + 담기 자동

```java
@GetMapping("/ex")
public void ex(@ModelAttribute("dto") SampleDto dto,
               @ModelAttribute("page") int page) {
    // 코드에 model.addAttribute 안 써도
    // dto → 모델 "dto", page → 모델 "page" 로 자동 등록
    // 뷰에서 ${dto}, ${page} 바로 사용 가능
}
```
- `@ModelAttribute("이름")`의 **이름**이 모델에 담길 때의 key가 된다.
- 즉 `@ModelAttribute("page") int page` 라고 쓰면 → 로직에서 `model.addAttribute("page", page)`를 **안 해도** 뷰로 바로 넘어간다.

---

## 4. 자동으로 모델에 담기는가? (타입에 따라 다름)

`@ModelAttribute`를 **안 붙였을 때**의 동작:

| 파라미터 타입 | 모델 자동 등록? | 뷰로 넘기려면 |
|------|:---:|------|
| **객체(DTO 등)** | ✅ 자동 (이름 = 클래스명 첫 글자 소문자) | 그냥 받으면 됨. 이름 바꾸려면 `@ModelAttribute("이름")` 또는 `addAttribute` |
| **기본형/String** (`int`, `String`…) | ❌ 안 됨 | `@ModelAttribute` 붙이거나 `model.addAttribute` 필요 |

```java
// SampleDto는 @ModelAttribute 없어도 "sampleDto" 이름으로 자동 등록됨
public void ex(SampleDto dto) { ... }     // 뷰에서 ${sampleDto}

// int는 자동 등록 안 됨 → 직접 처리해야 함
public void ex(@ModelAttribute("page") int page) { ... }   // 뷰에서 ${page}
```

---

## 5. 같은 결과, 두 가지 작성 방식

**방식 A — @ModelAttribute (자동)**
```java
public void ex(@ModelAttribute("dto") SampleDto dto,
               @ModelAttribute("page") int page) {
    // 별도 코드 없이 dto, page가 모델에 담김
}
```

**방식 B — 받아서 Model에 직접 담기**
```java
public void ex(SampleDto dto, @RequestParam int page, Model model) {
    model.addAttribute("dto", dto);
    model.addAttribute("page", page);
}
```
→ **결과 동일** (`${dto}`, `${page}` 사용 가능). A가 더 간결, B는 명시적/유연.

---

## 6. Model 파라미터를 안 받으면?

| 상황 | `@ModelAttribute` 값 | `addAttribute`로 직접 추가 |
|------|:---:|:---:|
| Model 파라미터 받음 | 뷰로 전달 ✅ | 가능 ✅ |
| Model 파라미터 안 받음 | **뷰로 전달 ✅** | 불가 ❌ |

> 정리: `@ModelAttribute`가 알아서 모델에 넣어주므로, 그 값들만 넘길 거면 `Model` 파라미터는 없어도 된다. 메서드 안에서 **추가** 데이터를 직접 담아야 할 때만 `Model`을 받는다.

---

## 한눈 요약
- **Model** = 뷰로 데이터 넘기는 상자 (항상 내부 존재, 파라미터는 손잡이)
- **@ModelAttribute** = 받기 + 담기 자동 (`("이름")`이 모델 key)
- **@RequestParam** = 받기만 / **model.addAttribute** = 담기만
- 객체(DTO)는 자동으로 담김(이름=클래스명 소문자), **단순 타입은 직접** 처리 필요
- `@ModelAttribute("page") int page` → 로직에 `addAttribute` 없이 `${page}`로 바로 사용

---

## 7. forward vs redirect (먼저 알아야 RedirectAttributes 이해됨)

컨트롤러가 끝나고 화면으로 갈 때, 두 가지 방식이 있다.

### forward (기본 — 지금까지의 방식)
- 서버 **내부에서** 다른 자원(JSP)으로 넘긴다. 클라이언트는 모름.
- 요청(request) **하나가 그대로 유지**됨 → 그 안에 담긴 Model 데이터도 그대로 살아있다.
- 브라우저 **URL이 안 바뀐다.** (요청 1번)
```
브라우저 → /sample/ex → [컨트롤러 → JSP] → 응답
                         └ 같은 요청 안에서 처리(URL 그대로)
```

### redirect
- 서버가 브라우저에게 **"저 주소로 다시 요청해"** 라고 응답(302). 브라우저가 **새 요청**을 보낸다.
- 요청이 **2번** 일어남 → 첫 요청의 Model 데이터는 **두 번째 요청에서 사라진다.** (request가 새것이라서)
- 브라우저 **URL이 바뀐다.**
```
브라우저 → /sample/save → 서버: "302, /sample/list 로 가" → 브라우저 → /sample/list (새 요청)
                                                              └ 첫 요청 데이터 소멸
```

### 코드에서 지정
```java
return "sample/list";            // forward (뷰 이름) → ViewResolver가 JSP 찾음
return "redirect:/sample/list";  // redirect → 브라우저가 /sample/list 로 재요청
return "forward:/sample/list";   // forward 명시
```

### 언제 redirect 를 쓰나
- **POST 처리 후** (등록/수정/삭제) → redirect 권장. ("PRG 패턴": Post-Redirect-Get)
- 이유: forward로 응답하면 **새로고침 시 POST가 재전송**되어 중복 등록될 수 있음. redirect하면 새로고침해도 GET만 반복돼 안전.

---

## 8. RedirectAttributes — redirect 시 데이터 넘기기

문제: redirect는 **요청이 새로 생겨서 Model 데이터가 사라진다.** 그럼 redirect 후 화면에 "저장됐습니다" 같은 값을 어떻게 넘기나? → `RedirectAttributes`를 쓴다.

```java
@PostMapping("/save")
public String save(SampleDto dto, RedirectAttributes rttr) {
    // ...저장 로직...
    rttr.addAttribute("id", dto.getId());        // ① URL 쿼리스트링으로
    rttr.addFlashAttribute("msg", "저장 완료");   // ② 플래시(1회용)로
    return "redirect:/sample/read";
}
```

### 두 가지 메서드 차이

| 메서드 | 전달 방식 | URL 노출 | 수명 |
|------|----------|:---:|------|
| `addAttribute("id", 10)` | **쿼리스트링**으로 붙음 → `?id=10` | 보임 ✅ | URL에 남아있는 동안 |
| `addFlashAttribute("msg", ...)` | **세션에 잠깐 저장** 후 redirect된 요청에서 한 번 꺼내고 사라짐 | 안 보임 ❌ | **1회용** (다음 요청 1번만) |

- `addAttribute`: redirect 대상 URL에 `?id=10` 형태로 붙는다. 그쪽 컨트롤러에서 `@RequestParam`/`@PathVariable`로 받음. (id 같은 식별값에 적합)
- `addFlashAttribute`: URL에 안 드러나고, redirect된 화면에서 `${msg}`로 한 번만 사용 가능. (성공 메시지 등 일회성 알림에 적합) → 새로고침하면 사라짐.

### ① addAttribute — URL 쿼리스트링으로 붙는다
```java
rttr.addAttribute("id", dto.getId());   // 예: id가 10
```
redirect되는 **URL 뒤에 `?id=10`이 자동으로 붙는다.**
```
return "redirect:/sample/read"
        ↓ 실제로 브라우저가 가는 주소
   /sample/read?id=10        ← addAttribute가 URL에 붙임
```
- **URL에 그대로 보인다** (주소창에 `?id=10` 노출).
- 받는 쪽에서 `@RequestParam`으로 꺼낸다:
  ```java
  @GetMapping("/read")
  public void read(@RequestParam Long id, Model model) { ... }  // id = 10
  ```
- **언제?** redirect 후에도 계속 필요한 **식별값**(어떤 글을 보여줄지 id). 새로고침해도 URL에 남아 다시 조회됨.

### ② addFlashAttribute — 잠깐 숨겨서 1번만 전달
```java
rttr.addFlashAttribute("msg", "저장 완료");
```
URL에 안 붙고, **서버 세션에 잠깐 저장**됐다가 redirect된 화면에서 **딱 한 번** 꺼내지고 사라진다.
```
/save → (msg를 세션에 잠깐 저장) → redirect → /sample/read
                                              ↓ 여기서 msg 꺼내 씀 → 즉시 삭제
                                         JSP에서 ${msg} → "저장 완료"
```
- **URL에 안 보인다** (주소창 깨끗).
- 받는 쪽에서 따로 받을 필요 없이 **뷰에서 바로 `${msg}`** 사용.
- **1회용**: 그 화면을 **새로고침하면 사라진다** (이미 한 번 꺼내 써서).
- **언제?** "저장 완료", "삭제되었습니다" 같은 **일회성 알림 메시지**. URL에 남으면 안 되고, 새로고침 때 또 뜨면 어색한 것들.

### 한 줄 비유
- `addAttribute` = **명찰 달아 보내기** — 누구나 보이는 곳(URL)에 붙여 보냄, 계속 남음
- `addFlashAttribute` = **쪽지 몰래 전달** — 안 보이게 한 번 전해주고, 받으면 바로 없어짐

> 그래서 예시 코드는: `id`는 "어떤 데이터를 보여줄지" 계속 필요하니 URL로(`addAttribute`), `"저장 완료"`는 한 번 띄우고 말 알림이니 플래시로(`addFlashAttribute`) — **용도에 맞게 나눠 쓴** 것이다.

---

## 9. Model 관련 컨트롤러 파라미터 정리

컨트롤러 메서드가 받을 수 있는, 데이터 전달 관련 주요 파라미터들.

| 파라미터/어노테이션 | 역할 | 비고 |
|------|------|------|
| `Model` | 뷰로 데이터 담기 (forward) | `addAttribute` |
| `@ModelAttribute` | 요청값 받기 + 모델에 담기 (자동) | `("이름")`이 key |
| `@RequestParam` | 요청 파라미터 1개 받기 | 담기는 안 함 |
| `@PathVariable` | URL 경로의 일부 받기 (`/read/{id}`) | |
| `@RequestBody` | 요청 본문(JSON 등) → 객체 | REST에서 |
| `RedirectAttributes` | **redirect 시** 데이터 전달 | `addAttribute`/`addFlashAttribute` |
| `HttpServletRequest/Response` | 서블릿 원본 객체 직접 접근 | 필요할 때만 |
| `HttpSession` | 세션 접근 | 로그인 등 |

> 정리: **forward면 `Model`**, **redirect면 `RedirectAttributes`**. redirect는 요청이 새로 생겨 Model이 안 통하므로 별도 도구(RedirectAttributes)가 필요하다는 게 핵심.

---

## 최종 요약
- **forward** = 서버 내부 이동, 요청 1개 유지(Model 살아있음), URL 그대로
- **redirect** = 브라우저 재요청, 요청 2개(Model 소멸), URL 바뀜, POST 후 권장(PRG)
- **Model** → forward용 데이터 전달 / **RedirectAttributes** → redirect용 데이터 전달
- `addAttribute`(쿼리스트링, 노출) vs `addFlashAttribute`(1회용, 비노출)