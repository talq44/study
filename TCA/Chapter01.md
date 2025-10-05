 # Chapter 1. Introduction & Motivation

 ## 왜 새로운 아키텍처가 필요한가?
 - 앱이 커지면 상태(state) 와 부수효과(side effect) 가 얽히면서 복잡성이 폭발
 - 전통적인 MVC, MVVM 은 처음엔 단순해 보이지만:
   - ViewController가 비대해짐 (Massive VC)
   - 테스트하기 어려움
   - 상태와 사이드이펙트가 분리되지 않음
 - Redux-style 아키텍처는 단방향 데이터 플로우로 복잡성 해결을 시도했지만 SwiftUI와 자연스럽게 맞물리지는 않음

👉 그래서 Point-Free는 Swift 언어 특성에 맞게 최적화된 단방향 아키텍처 = The Composable Architecture (TCA) 를 제안

---
## TCA의 핵심 목표
1. 단순성 (Simplicity)
   1. State, Action, Reducer, Environment 4가지 개념만으로 전체 앱을 설명할 수 있음
2. 테스트 용이성 (Testability)
   1. Reducer는 순수 함수 → Input(Action), Output(State)만으로 테스트 가능
3. 조합성 (Composability)
   1. 작은 기능을 모듈로 쪼개고, 다시 합쳐서 큰 앱을 만들 수 있음
4. 일관성 (Consistency)
   1. SwiftUI의 선언적 데이터 플로우와 자연스럽게 맞물림

---

## TCA의 4대 요소
> TCA는 단 4가지 타입만 이해하면 전부 다룰 수 있습니다.

1. State
   - 앱의 데이터 모델
   - Swift struct로 표현
    ```swift
    struct CounterState {
       var count = 0
    }
    ```

2. Action
   - 상태를 변화시키는 이벤트
   - Swift enum으로 표현
  
    ```swift
    enum CounterAction {
        case increment
        case decrement
    }
    ```

3. Reducer
   - Action이 들어왔을 때 State를 어떻게 바꿀지 정의하는 순수 함수
  
    ```swift
    let counterReducer = Reducer<CounterState, CounterAction, Void> { state, action, _ in
        switch action {
        case .increment:
            state.count += 1
            return .none
        case .decrement:
            state.count -= 1
            return .none
        }
    }
    ```

4. Environment
    - 외부 의존성 (API, DB, 타이머 등)을 주입하는 역할
    - 테스트 시 가짜 구현(mock)을 쉽게 넣을 수 있음
    ```swift
    struct CounterEnvironment { }
    ```

---

## SwiftUI와의 연결: Store
- 모든 것을 묶는 게 Store
- View는 Store에서 State를 구독하고, Action을 보냄
```swift
struct CounterView: View {
    let store: Store<CounterState, CounterAction>

    var body: some View {
        WithViewStore(self.store) { viewStore in
            VStack {
                Text("\(viewStore.count)")
                HStack {
                    Button("−") { viewStore.send(.decrement) }
                    Button("+") { viewStore.send(.increment) }
                }
            }
        }
    }
}
```

---

## Chapter 1 정리
- TCA는 단순한 규칙 4개(State, Action, Reducer, Environment) 로 앱을 구성
- MVC/MVVM보다 테스트 가능성, 모듈성, 일관성이 뛰어남
- SwiftUI의 선언적 UI와 잘 어울림
- 앞으로의 학습에서 이 Counter 예제를 시작으로 → Effects, Composability, Testing 등 확장

## 최신 버전 기준 보완 사항
- **`ReducerProtocol` 사용**: 최근 TCA에서는 `ReducerProtocol`을 통해 `State`와 `Action`을 reducer 내부에서 정의하는 방식이 기본임.
- **`Environment` → `Dependencies`**: 이제는 `Environment` 대신 `@Dependency`와 `DependencyValues`로 외부 의존성을 주입하는 방식이 권장됨.
- **`Effect` 처리**: Combine 기반뿐 아니라 async/await을 지원하는 `.task`, `.run` API 중심으로 업데이트됨.
- **네비게이션/알럿**: 최신 버전에서는 `@PresentationState`, `@PresentationAction` 등 속성 래퍼를 활용하는 방식이 추가됨.
- **Testing**: `TestStore`는 여전히 중심이지만, 최신 문법에서는 의존성 오버라이드와 시간 제어 기능이 강화됨.

## 📌 Chapter 1 질문 & 답변
### ❓ 1. 왜 MVVM 대신 TCA를 써야 하나요?

- 간단 답변
  - MVVM은 규모가 커질수록 ViewModel이 비대해지고 테스트하기 어려워집니다.
  - TCA는 State/Action/Reducer로 역할이 명확히 분리돼 있고, 테스트 가능성이 내장돼 있습니다.

- 심화 답변
  - MVVM은 “데이터 바인딩” 자체는 편하지만, 비즈니스 로직과 사이드 이펙트가 ViewModel 내부에서 섞여 버리기 쉽습니다.
  - TCA는 Reducer가 순수 함수라 테스트가 쉬우며, Effect를 통해 사이드 이펙트를 명확히 분리합니다.
  - 또한 작은 Feature 단위로 모듈화(pullback, combine) 할 수 있어, 대규모 앱 확장성에서 MVVM보다 유리합니다.

### ❓ 2. Reducer가 “순수 함수”라는 게 왜 중요한가요?

- 간단 답변
  - 입력(State, Action)이 같으면 항상 같은 출력(State)을 보장합니다.
  - 따라서 예측 가능하고, 테스트하기 쉽습니다.

- 심화 답변
  - 순수 함수(Pure function)는 외부 상태나 사이드 이펙트에 의존하지 않으므로 디버깅과 리팩토링이 안전합니다.
    - 예: counterReducer는 .increment가 들어오면 항상 count + 1이라는 결과를 냅니다.
  - 테스트에서는 단순히 Reducer에 State/Action을 넣고, 예상한 State가 나오는지만 검증하면 됩니다.

### ❓ 3. State와 Environment는 어떤 기준으로 나누나요?

- 간단 답변
    - State: 앱 내부에서 유지해야 하는 데이터
    - Environment: 외부 세계(API, DB, Timer 등)와 연결되는 의존성

- 심화 답변
    - State는 UI와 직접 연결되는 값이고, Environment는 변경 가능성이 크고 외부 시스템에 의존하는 요소를 담습니다.
      - 예: Counter 앱
      - State: 현재 카운트 값(Int)
      - Environment: 타이머, 랜덤 숫자 생성기, 네트워크 클라이언트
    - 이렇게 구분하면 State는 항상 순수하게 테스트할 수 있고, Environment는 mock/stub으로 대체해 테스트가 가능해집니다.

### ❓ 4. Store가 없으면 SwiftUI State만으로도 구현 가능한데, 차이가 뭔가요?

- 간단 답변
  - SwiftUI의 @State/@ObservedObject는 작은 화면에는 충분합니다.
  - 그러나 앱이 커질수록 상태 공유, 모듈성, 테스트 가능성이 부족합니다.

- 심화 답변
  - @State/@ObservedObject는 View 계층 안에 묶여 있어서 재사용과 모듈화에 제약이 있습니다.
  - Store는 상태 관리와 액션 처리를 View로부터 완전히 분리해서, UI 없는 테스트도 가능합니다.
  - 또한 Store는 Reducer, Effect, Composability와 자연스럽게 연결돼 대규모 앱에서 강력해집니다.
  - 즉, SwiftUI의 단순 상태 관리 + Redux-style 설계 = Store

### ✅ 정리
- MVVM보다 TCA가 나은 이유: 확장성과 테스트 용이성
- Reducer의 순수성: 예측 가능성 + 테스트 가능성
- State vs Environment: 내부 데이터 vs 외부 의존성
- Store vs SwiftUI State: 소규모엔 State, 대규모엔 Store