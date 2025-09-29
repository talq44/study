# Chapter 6. Composability

## Composability란?
- 앱은 여러 개의 작은 기능(Feature)으로 구성됨.
- 작은 Reducer 들을 큰 Reducer 로 합치는 것이 핵심.
- Store도 동일하게 Scope를 통해 부분 Store로 나눠서 전달 가능.

## Reducer 합성
### 예전 방식 (combine, pullback)
```swift
let appReducer = Reducer<AppState, AppAction, AppEnvironment>.combine(
  counterReducer.pullback(
    state: \.counter,
    action: /AppAction.counter,
    environment: { $0.counter }
  ),
  profileReducer.pullback(
    state: \.profile,
    action: /AppAction.profile,
    environment: { $0.profile }
  )
)
```

---
```swift
struct AppFeature: Reducer {
  struct State: Equatable {
    var counter = CounterFeature.State()
    var profile = ProfileFeature.State()
  }

  enum Action: Equatable {
    case counter(CounterFeature.Action)
    case profile(ProfileFeature.Action)
  }

  var body: some ReducerOf<Self> {
    Scope(state: \.counter, action: /Action.counter) {
      CounterFeature()
    }
    Scope(state: \.profile, action: /Action.profile) {
      ProfileFeature()
    }
  }
}
```
- Scope : 상위 Feature의 State/Action 일부를 하위 Feature와 연결
- 이 방식은 더 읽기 쉽고, SwiftUI와 자연스럽게 통합됨

## Store 합성
- 상위 Store에서 하위 Store를 만들어 전달:
```swift
struct AppView: View {
  let store: StoreOf<AppFeature>

  var body: some View {
    VStack {
      Scope(state: \.counter, action: AppFeature.Action.counter) {
        CounterView(store: $0)
      }
      Scope(state: \.profile, action: AppFeature.Action.profile) {
        ProfileView(store: $0)
      }
    }
  }
}
```
- Store를 전역으로 공유하지 않고, 필요한 부분만 잘라서 자식 View에 전달하는 방식

## ForEach Reducer
- 배열/컬렉션 기반 State를 관리할 때는 ForEachReducer 활용:
```swift
struct TodosFeature: Reducer {
  struct State: Equatable {
    var todos: IdentifiedArrayOf<TodoFeature.State> = []
  }
  enum Action: Equatable {
    case todos(id: TodoFeature.State.ID, action: TodoFeature.Action)
  }

  var body: some ReducerOf<Self> {
    ForEach(\.todos, action: /Action.todos) {
      TodoFeature()
    }
  }
}
```
- SwiftUI의 ForEach와 자연스럽게 매칭 → 각 item이 독립적인 Store/Reducer를 가짐

## Optional/Enum State 관리
- Feature가 선택적(Optional)로 존재하거나, enum case로 상태가 전환될 때는 ifLet, ifCaseLet 사용:
```swift
struct AppFeature: Reducer {
  struct State: Equatable {
    var settings: SettingsFeature.State?
  }
  enum Action: Equatable {
    case settings(SettingsFeature.Action)
  }

  var body: some ReducerOf<Self> {
    ifLet(\.settings, action: /Action.settings) {
      SettingsFeature()
    }
  }
}
```

## 꼬리 질문 & 답변 예시
- ❓ Scope가 왜 중요한가요?
  - 간단 답변: 상위 Store에서 필요한 부분만 잘라서 자식 Store에 전달하기 위해.
  - 심화 답변: 전역 Store 공유를 피하고, Feature 단위로 상태/액션을 분리해 테스트 용이성과 모듈성을 확보.

- ❓ ForEachReducer는 언제 쓰나요?
  - 간단 답변: State가 배열/컬렉션 형태일 때.
  - 심화 답변: 각 item이 독립적인 Reducer/State를 가질 수 있어, 재사용성과 테스트 용이성이 높아짐.

- ❓ ifLet과 ifCaseLet의 차이는?
  - ifLet: Optional State를 unwrap 해서 Reducer를 실행.
  - ifCaseLet: Enum의 특정 case일 때 Reducer 실행.

- ❓ combine / pullback은 이제 안 쓰나요?
  - 간단 답변: 여전히 존재하지만, 최신 스타일은 ReducerProtocol + body + Scope/ForEach/ifLet 권장.
  - 심화 답변: 새로운 방식은 Swift 문법과 잘 맞고, boilerplate가 줄어듦.

## Chapter 6 요약
- 작은 Reducer → 큰 Reducer로 합치는 것이 Composability
- 최신 방식은 ReducerProtocol + Scope, ForEach, ifLet 활용
- Store도 부분 Store를 잘라내 전달하는 방식이 핵심
- 전역 Store 공유는 지양하고, Feature 단위로 상태/액션을 모듈화

## 참고 페이지
- https://axiomatic-fuschia-666.notion.site/Chapter-7-MultiStore-1e8734581d104073b9d0ec3697f5ee1d