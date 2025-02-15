### 💡들어가기

- 해당 주제에 대한 선정이유 (순수하게 궁금해서 이런것도 OK)
    - Flow는 직접 생명주기에 맞춰서 사용해야 하는데 Compose에서는 과연 어떻게 해야 안전하게 사용할까! 싶어서 가져와봤습니다
- 참고자료 레퍼런스
    - https://medium.com/androiddevelopers/consuming-flows-safely-in-jetpack-compose-cde014d0d5a3
    - https://developer.android.com/develop/ui/compose/lifecycle
  
<br></br>

Compose는 Recomposition이 발생할 수 있기 때문에, Flow를 직접 collect하면 불필요한 중복 구독이 발생할 수 있다

## 그렇다면 Compose에서 Flow를 안전하게 사용하려면 어떻게 해야할까 🤔

<br></br>

## 1️⃣ Flow를 State로 변환하여 UI에서 사용하기 : `collectAsStateWithLifecycle`

Compose에서 Flow를 안전하게 구독하려면 Flow를 State로 변환해야 함

→ `collectAsStateWithLifecycle()` 사용

```kotlin
val state by viewModel.state.collectAsStateWithLifecycle()
```
<br></br>
`collectAsStateWithLifecycle()`은 flow에서 값을 수집하고 최신 값을 Compose state로 나타내는 라이프 사이클 인식 방식의 composable 함수

- 기본적으로 `collectAsStateWithLifecycle`은 `Lifecycle.State.STARTED`를 사용해 flow에서 값 수집을 시작하거나 중지함

<br></br>

### 그렇다면 왜 collectState() 보다 collectAsStateWithLifecycle()이 좋을까?

![image](https://github.com/user-attachments/assets/4f6c0ee9-5690-48fc-ba14-b98280e024be)

라이프 사이클에 맞추어 collectAsState와 collectAsStateWithLifecycle을 비교

( 제가 처음에 이 라이프사이클 표를 보고 collectAsStateWithLifecycle이 액티비티의 라이프사이클을 따른다고 생각했는데요, 😅
다른 방법(?)이 또 있더라구요..!! 이 부분은 아래 3번에 다시 작성해두었습니다
위 사진은 그냥 collectAsStateWithLifecycle과 collectAsState의 차이를 위한 용도로 이해해주시면 좋을 것 같아요 ! )

- `collectAsStateWithLifecycle`은 앱이 백그라운드로 진입했을 때 flow 수집을 멈추고 다시 포그라운드로 돌아왔을 때 재개
- 반면, `collectAsState`는 백그라운드로 진입하여도 수집 작업이 활성화된 상태이며, 라이프 사이클이 타겟 상태가 바깥으로 변경될 경우 문제가 발생할 수 있음

<br></br>

## 2️⃣ 그럼 collectAsState()는 언제, 왜 사용할까?

`collectAsState`는 Composition의 라이프사이클을 따르고, composable이 Composition 단계에 진입할 때 flow를 수집하기 시작 !

→ `collectAsState`는 앱이 백그라운드에 있는 동안 recomposition을 중지하더라도 수집 작업이 활성화된 상태로 유지하기 때문에 flow를 수집하는데 플랫폼에 구애받지 않는 API
<br></br>

그렇기 때문에 `collectAsStateWithLifecycle`은 Android의 lifecycle에 맞춰서 앱을 개발할 때,

`collectAsState`는 다른 플랫폼을 위해 개발할 때 사용 (플랫폼 의존성을 없애야 할 때)

(준비하면서 다양한 블로그를 읽어보니 compose 사용 시 주의할 것 중에 ***‘Flow를 collectAsState()를 통해 소비하는 것’***이 있더라구요
보통 안드로이드 개발에서 collectAsState()는 잘 사용하지 않는듯 합니다..!!)

<br></br>

## 3️⃣ 그럼 Single Activity Architecture에서는 앱이 켜져있는 동안 계속 추적할까?

collectAsStateWithLifecycle이 따르는 라이프 사이클은 경우에 따라 2가지로 나뉜다고 하는데요..!

<br></br>

### 1. Navigation을 사용한 화면(Screen) 전환 시 (`NavHost` 이용)

`NavBackStackEntry`가 `LifecycleOwner` 역할을 함

쉽게 말해서 액티비티의 생명주기를 따르는 것이 아니라 각 Composable의 생명주기를 따라게 됨 !

즉, 각 화면의 라이프사이클을 따르게 되는 것 !!
<br></br>

해당 화면이 활성화(Lifecycle.State.STARTED) 상태일 때만 Flow를 수집

화면이 사라지면(Lifecycle.State.STOPPED) 상태 시 Flow 수집 중단

<br></br>

잠깐 Composable의 라이프 사이클을 알아보고 가자면 ~

Composable의 라이프 사이클은 액티비티의 라이프 사이클만큼 복잡하진 않고 3가지로 나뉩니다

![image](https://github.com/user-attachments/assets/b6268f23-917e-4450-80f1-4994212faf7b)

1. Enter the Composition (초기 Composition)
    - Composable이 처음 Composition에 추가됨
    - STARTED 상태 → Flow 수집
2. Recompose 0 or more times
    - State가 변경되어 UI가 다시 그려짐
    - STARTED 상태 → Flow 수집
3. Leave the Composition (Composition에서 제거됨)
    - Composable이 Composition에서 제거됨
    - STOPPED 상태 → Flow 수집 X

[Composable 라이프사이클 관련 공식문서]

[Lifecycle of composables  |  Jetpack Compose  |  Android Developers](https://developer.android.com/develop/ui/compose/lifecycle)

<br></br>

### 2. Navigation을 사용하지 않고 단순 Composable 교체 시

이 경우엔 저희가 처음 이야기 나눴던 것처럼 Single Activity를 사용한다면 `LifecycleOwner`가 Activity가 되어 액티비티 라이프 사이클을 따름

그럼 그때 저희가 고려했던 내용처럼 Flow가 계속 수집이 될 위험이 있겠죠 !?
<br></br>

만약 이렇게 Navigation 없이 Composable을 교체하는 경우에는 rememberSaveable()과 DisposableEffect를 활용해서 Lifecycle을 수동으로 관리

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val lifecycleOwner = LocalLifecycleOwner.current
    val state by viewModel.someFlow.collectAsStateWithLifecycle(lifecycleOwner = lifecycleOwner)
}
```

