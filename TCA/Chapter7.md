# Chapter 7. Testing

## 왜 TCA는 테스트하기 쉬운가?
- Reducer는 순수 함수 → Input(Action, State) → Output(State, Effect)만으로 동작
- Effect는 명확하게 분리되어 있고, 의존성 주입(@Dependency) 으로 Mock/Test 구현을 넣을 수 있음
- 따라서 UI 없이도 비즈니스 로직만 테스트 가능

## TestStore
- TCA에서 테스트의 핵심은 TestStore
- 실제 Store와 비슷하지만, Reducer/State/Action의 입출력을 단계별로 검증할 수 있게 도움
```swift
let store = TestStore(
  initialState: CounterFeature.State(),
  reducer: CounterFeature()
)

await store.send(.increment) {
  $0.count = 1   // 예상되는 상태 변화
}

await store.send(.decrement) {
  $0.count = 0
}
```
- send: 액션을 보냄
- 클로저 내부에서 예상되는 상태 변화를 검증

## Effect 테스트 (비동기)
```swift
@Dependency(\.apiClient) var apiClient

// Reducer
case .loadData:
  state.isLoading = true
  return .run { send in
    let items = try await apiClient.fetchItems()
    await send(.itemsResponse(items))
  }
```
- 테스트에서는 apiClient.fetchItems를 Mock으로 교체:
```swift
let store = TestStore(
  initialState: Feature.State(),
  reducer: Feature()
) {
  $0.apiClient.fetchItems = { ["item1", "item2"] }
}

await store.send(.loadData) {
  $0.isLoading = true
}

await store.receive(.itemsResponse(["item1", "item2"])) {
  $0.isLoading = false
  $0.items = ["item1", "item2"]
}
```

## Cancellation 테스트
- Timer, 네트워크 요청 등이 취소되는지 확인 가능
```swift
await store.send(.startTimer) {
  $0.isTimerOn = true
}
await store.send(.stopTimer) {
  $0.isTimerOn = false
}
```
- 취소 후에는 Effect가 더 이상 Action을 방출하지 않음

## Scheduler / 시간 제어
- TCA는 TestScheduler 또는 TestClock을 제공
- 시간 흐름을 제어하며 테스트 가능
```swift
@Dependency(\.continuousClock) var clock

// Test
let clock = TestClock()
let store = TestStore(
  initialState: TimerFeature.State(),
  reducer: TimerFeature()
) {
  $0.continuousClock = clock
}

await store.send(.startTimer)
await clock.advance(by: .seconds(1))
await store.receive(.tick)
```

## 최신 버전 보완 사항
- 예전에는 Environment override 방식이었으나,
  - → 지금은 DependencyValues + @Dependency 를 테스트에서 오버라이드하는 게 표준
- TestStore는 이제 async/await 기반 → await store.send, await store.receive
- Non-exhaustive 테스트도 가능: 모든 Action을 다 검증하지 않아도 됨
- Preview와 Test에서 동일한 방식으로 Dependency override가 가능
  - TMA 구조에서도 활용 가능

## 꼬리 질문 & 답변 예시
- ❓ TCA 테스트가 다른 아키텍처보다 쉬운 이유는?
  - 간단 답변: Reducer가 순수 함수라서.
  - 심화 답변: Action → State 변화를 함수로 분리했기 때문에 UI/플랫폼에 의존하지 않고 검증 가능. 의존성도 주입/오버라이드 방식이라 Mock을 쉽게 넣을 수 있음.

- ❓ Effect 테스트는 어떻게 하나요?
  - 간단 답변: TestStore의 receive를 사용해 기대되는 Action을 확인.
  - 심화 답변: Effect 내부에서 Action을 send하도록 구성하기 때문에, 테스트에서 해당 Action이 도착하는지, State가 맞게 바뀌는지를 확인하면 됨.

- ❓ Scheduler / Clock을 테스트에 쓰는 이유는?
  - 간단 답변: 시간을 제어하기 위해.
  - 심화 답변: Timer, Delay, Debounce 같은 비동기 작업은 실제 시간 흐름에 의존하면 테스트가 느려지고 불안정. TestClock을 사용하면 원하는 시점에 advance(by:)를 호출해 정확히 제어 가능.

## Chapter 7 요약
- TestStore = TCA 테스트의 핵심
- 순수 Reducer + 의존성 주입 → Mock 테스트 쉬움
- async/await 기반 Effect 테스트, Cancellation, Scheduler 제어 가능
- 최신 버전에서는 DependencyValues 오버라이드로 테스트 구현

## 참고 링크
- https://axiomatic-fuschia-666.notion.site/Chapter-9-TCA-Testable-Code-ad3924113fbb4f89a06f30ddb8e884f7