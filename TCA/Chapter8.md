# Chapter 8. Ergonomics

## BindingState와 BindingAction
- BindingState: SwiftUI의 @State와 동일한 역할을 TCA의 State에서 수행.
- BindingAction: SwiftUI의 Binding 이벤트를 Reducer로 전달.
- 함께 쓰면 View에서 @Binding처럼 직접 바인딩 가능.

```swift
struct ProfileFeature: Reducer {
  struct State: Equatable {
    @BindingState var username: String = ""
    @BindingState var isPrivate: Bool = false
  }

  enum Action: BindableAction, Equatable {
    case binding(BindingAction<State>)
    case saveButtonTapped
  }

  var body: some ReducerOf<Self> {
    BindingReducer()
    Reduce { state, action in
      switch action {
      case .binding:
        // BindingReducer에 의해 처리되므로 이 케이스는 실행되지 않지만,
        // 컴파일러의 switch문 완전성 검사를 통과하기 위해 필요합니다.
        return .none
      case .saveButtonTapped:
        print("Saved: \(state.username), private: \(state.isPrivate)")
        return .none
      }
    }
  }
}
```

## SwiftUI 연결
```swift
struct ProfileView: View {
  let store: StoreOf<ProfileFeature>

  var body: some View {
    WithViewStore(self.store, observe: { $0 }) { viewStore in
      Form {
        TextField("Username", text: viewStore.binding(\.$username))
        Toggle("Private", isOn: viewStore.binding(\.$isPrivate))
        Button("Save") { viewStore.send(.saveButtonTapped) }
      }
    }
  }
}
```

- viewStore.binding(\.$property)로 View와 Reducer를 양방향으로 연결
- SwiftUI의 @Binding과 거의 동일한 문법

## BindingReducer()
- BindingReducer()는 BindingAction을 자동으로 처리해줌.
- .binding 케이스를 직접 switch에 추가하지 않아도 됨.
- 여러 Feature에서도 공통적으로 사용 가능.

## 최신 버전 기준 (1.x) 개선 사항
항목 | 최신 개선 내용
--|--
BindingReducer() | 별도 선언 없이도 Reducer 내부에 자연스럽게 통합 가능 (Result Builder 문법으로 body 안에 포함 가능)
@BindableState → @BindingState | 이름이 명확해지고, SwiftUI의 @Binding과 일관성 향상
ViewStore.binding(_:) | SwiftUI Binding을 생성하는 공식 API로 정착
@PresentationState, @PresentationAction | Navigation/Sheet/Alert 도 Binding처럼 선언적으로 관리 가능

## Navigation / Sheet / Alert 연동 (간략 예시)
```swift
struct SettingsFeature: Reducer {
  struct State: Equatable {
    @PresentationState var destination: Destination.State?
  }

  enum Action: Equatable {
    case destination(PresentationAction<Destination.Action>)
    case helpButtonTapped
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .helpButtonTapped:
        state.destination = .help(HelpFeature.State())
        return .none
      case .destination:
        return .none
      }
    }
    .ifLet(\.$destination, action: /Action.destination) {
      Destination()
    }
  }

  struct Destination: Reducer {
    enum State: Equatable { case help(HelpFeature.State) }
    enum Action: Equatable { case help(HelpFeature.Action) }

    var body: some ReducerOf<Self> {
      Scope(state: /State.help, action: /Action.help) {
        HelpFeature()
      }
    }
  }
}
```

- SwiftUI에서의 연결

```swift
struct SettingsView: View {
  let store: StoreOf<SettingsFeature>

  var body: some View {
    WithViewStore(store, observe: { $0 }) { viewStore in
      Button("Help") { viewStore.send(.helpButtonTapped) }
        .sheet(store: store.scope(state: \.$destination, action: SettingsFeature.Action.destination))
    }
  }
}
```
- 이 패턴 덕분에 SwiftUI의 네비게이션 계층도 완전히 선언형으로 관리 가능.

## 꼬리 질문 & 답변 예시
- ❓ BindingState와 @State의 차이는?
  - 간단 답변: BindingState는 Reducer가 제어하는 상태, @State는 View 내부 로컬 상태.
  - 심화 답변: BindingState는 단방향 데이터 플로우를 유지하면서도 SwiftUI의 양방향 입력을 지원. 
  - 즉, View의 변경사항이 Reducer의 Action으로 변환되어 상태 일관성을 유지함.

- ❓ BindingReducer는 왜 필요한가요?
  - 간단 답변: BindingAction을 자동 처리해주기 때문.
  - 심화 답변: BindingReducer가 없으면 모든 입력마다 수동으로 .binding case를 처리해야 함. Boilerplate를 제거하고 선언적으로 Reducer를 구성하게 해줌.

- ❓ PresentationState / Action은 뭐가 다른가요?
  - 간단 답변: Navigation/Alert을 BindingState처럼 선언적으로 관리하기 위한 도구.
  - 심화 답변: Sheet나 NavigationLink를 imperative하게 띄우는 대신, State 변화에 따라 자동으로 표시/해제됨. SwiftUI와 완전히 일관된 데이터 흐름을 유지할 수 있음.

## Chapter 8 요약
- SwiftUI의 바인딩을 TCA의 단방향 구조로 통합
- @BindingState, BindingAction, BindingReducer() → 양방향 데이터 처리 간소화
- Navigation/Sheet/Alert도 @PresentationState 기반으로 선언적 관리 가능
- UI 이벤트를 Reducer에서 완전히 제어 가능

## 참고
- https://axiomatic-fuschia-666.notion.site/Chapter-4-TCA-Binding-87f748b3f8fa41a08a9089d78aeb422c