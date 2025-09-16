 # Introduction & Motivation

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

