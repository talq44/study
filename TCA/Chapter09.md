# Chapter 9. Navigation

## 기존 방식의 문제점
- 기존 SwiftUI 네비게이션은 @State로 Bool 플래그를 관리하는 경우가 많았음:
```swift
@State var isShowingSheet = false

Button("Show") {
  isShowingSheet = true
}
.sheet(isPresented: $isShowingSheet) {
  DetailView()
}
```

- ⚠️ 문제점
  - View 단에서 상태를 직접 관리 → Reducer(State)와 일관성 깨짐
  - 화면 닫힘/이동을 Reducer가 알 수 없음 (ex. dismiss 후 데이터 초기화 불가)

## 최신 TCA 네비게이션 패턴
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
    enum State: Equatable {
      case help(HelpFeature.State)
    }
    enum Action: Equatable {
      case help(HelpFeature.Action)
    }

    var body: some ReducerOf<Self> {
      Scope(state: /State.help, action: /Action.help) {
        HelpFeature()
      }
    }
  }
}
```
### SwiftUI 연결
```swift
struct SettingsView: View {
  let store: StoreOf<SettingsFeature>

  var body: some View {
    WithViewStore(store, observe: { $0 }) { viewStore in
      Button("Help") {
        viewStore.send(.helpButtonTapped)
      }
      .sheet(
        store: store.scope(state: \.$destination, action: SettingsFeature.Action.destination)
      ) { destStore in
        SwitchStore(destStore) {
          CaseLet(/SettingsFeature.Destination.State.help, action: SettingsFeature.Destination.Action.help) { helpStore in
            HelpView(store: helpStore)
          }
        }
      }
    }
  }
}
```
- @PresentationState : 모달/네비게이션 상태를 State로 표현
- store.scope(state: \.$destination, action: SettingsFeature.Action.destination) : View와 Reducer 연결
- SwitchStore : enum 기반 화면 전환 (Destination 패턴)

## NavigationStack 관리
- Push 네비게이션도 같은 방식으로 가능:
```swift
struct AppFeature: Reducer {
  struct State: Equatable {
    var path: StackState<Path.State> = .init()
  }
  enum Action: Equatable {
    case path(StackAction<Path.State, Path.Action>)
    case showDetail
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .showDetail:
        state.path.append(.detail(DetailFeature.State()))
        return .none
      case .path:
        return .none
      }
    }
    .forEach(\.path, action: /Action.path) {
      Path()
    }
  }

  struct Path: Reducer {
    enum State: Equatable {
      case detail(DetailFeature.State)
    }
    enum Action: Equatable {
      case detail(DetailFeature.Action)
    }

    var body: some ReducerOf<Self> {
      Scope(state: /State.detail, action: /Action.detail) {
        DetailFeature()
      }
    }
  }
}
```
- SwiftUI 연결 예시:
```swift
struct AppView: View {
  let store: StoreOf<AppFeature>

  var body: some View {
    NavigationStackStore(
      self.store.scope(state: \.path, action: AppFeature.Action.path)
    ) {
      WithViewStore(store, observe: { $0 }) { viewStore in
        Button("Go to Detail") {
          viewStore.send(.showDetail)
        }
      }
    } destination: {
      CaseLet(/AppFeature.Path.State.detail, action: AppFeature.Path.Action.detail) { store in
        DetailView(store: store)
      }
    }
  }
}
```

## 최신 버전(1.x)에서의 네비게이션 변화 요약
항목 | 설명
-- | --
@PresentationState | 모달, 시트, Alert 등 단일 프리젠테이션 상태를 표현 가능
@PresentationAction | UI 이벤트를 Reducer로 자동 전달
StackState / StackAction | NavigationStack 상태/이벤트 모델링
SwitchStore, CaseLet | Enum-based 화면 분기 처리
.ifLet, .forEach | Scope 기반 하위 Reducer 조합
NavigationStackStore, .sheet(store:), .alert(store:) | SwiftUI와 완전 통합된 선언적 Navigation

## 꼬리 질문 & 답변 예시
- ❓ 왜 View 대신 Reducer가 Navigation을 제어해야 하나요?
  - 간단 답변: 상태 일관성을 유지하기 위해.
  - 심화 답변: SwiftUI의 View 계층과 별도로 Navigation 상태를 Reducer가 관리하면, 화면 전환이 명확히 Action 기반으로 추적 가능하고, UI와 비즈니스 로직의 분리가 완전해집니다.

- ❓ NavigationStackStore와 일반 NavigationStack의 차이는?
  - 간단 답변: Stack 상태를 Reducer가 제어하느냐 여부.
  - 심화 답변: NavigationStackStore는 StackState를 기반으로 Push/Pop을 Action으로 관리하고, 테스트/프리뷰에서도 동일한 로직으로 실행 가능.

- ❓ @PresentationState가 중요한 이유는?
  - 간단 답변: 네비게이션을 State의 일부로 선언적으로 관리하기 위해.
  - 심화 답변: Sheet, Alert, Push 등이 UI 상태가 아니라 데이터의 파생으로 동작하게 되어, “View가 열려 있느냐 닫혔느냐”가 아니라 “State가 존재하느냐”로 제어됨.

## Chapter 9 요약
- Navigation은 이제 State 기반으로 완전히 선언적 관리
- @PresentationState, @PresentationAction, StackState, SwitchStore가 핵심
- View는 단순히 Store에 연결할 뿐, 화면 전환의 주체는 Reducer
- Push / Sheet / Alert / FullScreenCover 모두 같은 방식으로 통일

## 참고링크
- https://axiomatic-fuschia-666.notion.site/Chapter-8-Navigation-855a4db02ef346e5b6ff8c35c7db3096