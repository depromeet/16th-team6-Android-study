동영상이 잘 안나올 경우 아래 링크를 참고해주세요
https://www.notion.so/depromeet/3-Animation-6eb35f3713d84b1f99c75f5c8566ffd7

# **`animate*AsState`**

[`animate*AsState`](https://developer.android.com/reference/kotlin/androidx/compose/animation/core/package-summary?hl=ko#animateDpAsState(androidx.compose.ui.unit.Dp,androidx.compose.animation.core.AnimationSpec,kotlin.String,kotlin.Function1)) 함수는 Compose에서 단일 값을 애니메이션 처리하는 가장 간단한 애니메이션 API입니다. 타겟 값 (또는 최종 값)만 제공하면 API가 현재 값에서 지정된 값으로 애니메이션을 시작합니다.

즉, 어떤 값이 변경될 때 자동으로 애니메이션이 적용되도록 하는 API입니다.

주로 `size`, `color`, `alpha`, `offset` 등을 부드럽게 변경하는 데 사용합니다.

타겟 값을 [`animateFloatAsState`](https://developer.android.com/reference/kotlin/androidx/compose/animation/core/package-summary?hl=ko#animateFloatAsState(kotlin.Float,androidx.compose.animation.core.AnimationSpec,kotlin.Float,kotlin.String,kotlin.Function1))에 래핑하기만 하면 알파 값이 제공된 값 (이 경우 `1f` 또는 `0.5f`) 사이의 애니메이션 값이 됩니다.

```kotlin
var enabled by remember { mutableStateOf(true) }

val alpha: Float by animateFloatAsState(if (enabled) 1f else 0.5f, label = "alpha")
Box(
    Modifier
        .fillMaxSize()
        .graphicsLayer(alpha = alpha)
        .background(Color.Red)
)
```

```kotlin
@Composable
fun AnimatedSizeBox() {
    var isExpanded by remember { mutableStateOf(false) }
    
    val size by animateDpAsState(
        targetValue = if (isExpanded) 200.dp else 100.dp,
        animationSpec = tween(durationMillis = 500) // 0.5초 동안 변경
    )

    Box(
        modifier = Modifier
            .size(size)
            .background(Color.Blue)
            .clickable { isExpanded = !isExpanded }
    )
}

```

약간 애니메이션을 State처럼 쓰는 느낌?

[bandicam 2025-02-06 20-14-16-836.mp4](attachment:285f6bcb-3fe3-432b-91f1-16fe56631922:bandicam_2025-02-06_20-14-16-836.mp4)

# `AnimatedVisibility` (UI 요소의 등장/사라짐)

컴포넌트가 나타나거나 사라질 때 애니메이션을 적용할 수 있습니다.

```kotlin
@Composable
fun AnimatedVisibilityWithCustomAnimation() {
    var isVisible by remember { mutableStateOf(false) }
    Box(modifier = Modifier.fillMaxSize()) {
        AnimatedVisibility(
            visible = isVisible,
            modifier = Modifier.align(Alignment.TopCenter),
            enter = slideInVertically(initialOffsetY = {// it : y offset
                -it
            }),
            exit = slideOutVertically(targetOffsetY = {
                -it
            })
        ) {
            Box(
                modifier = Modifier
                    .size(100.dp)
                    .clip(CircleShape)
                    .background(Color.Blue)
            )
        }

        Button(
            modifier = Modifier
                .align(Alignment.BottomCenter)
                .fillMaxWidth()
                .padding(horizontal = 30.dp, vertical = 50.dp)
                .height(50.dp),
            onClick = {
                isVisible = isVisible.not()
            }
        ) {
            Text(text = "Visibility 변경하기")
        }
    }
}
```

- `AnimatedVisibility`를 사용하면 `visible` 값에 따라 자동으로 애니메이션이 적용됨.
- 내부적으로 `fade`, `scale`, `slide` 같은 효과를 추가할 수 있음.

[bandicam 2025-02-06 20-26-06-821.mp4](attachment:aef3531d-66d5-4d50-980a-ef2be43cb35f:bandicam_2025-02-06_20-26-06-821.mp4)

![image.png](attachment:056b256c-67b3-4625-9942-5dc7b9fab77a:image.png)

- visible : visible상태(State)가 변화할 때 Animation이 Trigger된다. visible 이 true일 때는 EnterTransition(애니메이션)이 수행되며, visible이 false일 때는 ExitTransition이 수행된다.
- modifier : AnimatedVisibliity는 애니메이션을 할 Content를 감싸는 레이아웃 역할을 한다. 따라서 modifier는 레이아웃의 modifier가 된다.
- enter : enter은 EnterTransition을 인자로 받는다. visible이 false에서 true로 바뀔 때 Trigger되는 애니메이션을 정의한다.
- exit : exit은 ExitTransition을 인자로 받는다. visible이 true에서 false로 바뀔 때 Trigger되는 애니메이션을 정의한다.
- label : label은 이 애니메이션이 무엇을 하는지 설명하기 위한 label이다. 중요하지 않다.
- content : content는 Animated Visibility를 적용할 Composable Content이다.

중요한건 enter와 exit의 transition을 어떻게 커스텀 하냐에 따라 다양한 애니메이션을 구현할 수 있다는 것입니다.

해당 transition은 EnterExitTransition 클래스에 정의되어 있는 매서드를 사용해야 하며 그 종류는 다음과 같이 공식문서에서 찾아볼 수 있습니다.

[애니메이션 수정자 및 컴포저블  |  Jetpack Compose  |  Android Developers](https://developer.android.com/develop/ui/compose/animation/composables-modifiers?hl=ko#enter-exit-transition)

# `updateTransition` (여러 속성을 동시에 애니메이션)

**여러 상태 간의 복잡한 애니메이션 전환**을 처리합니다. 

하나의 상태에 여러 애니메이션을 동시에 적용하거나 상태 변화에 따른 애니메이션 제어가 가능.

```kotlin
@Composable
fun TransitionExample() {
    var isExpanded by remember { mutableStateOf(false) }

    val transition = updateTransition(targetState = isExpanded, label = "box transition")

    val size by transition.animateDp(label = "size") { state ->
        if (state) 200.dp else 100.dp
    }

    val color by transition.animateColor(label = "color") { state ->
        if (state) Color.Red else Color.Blue
    }

    Box(
        modifier = Modifier
            .size(size)
            .background(color)
            .clickable { isExpanded = !isExpanded }
    )
}
```

- `updateTransition`을 사용하면 `size`와 `color`가 동시에 애니메이션됩니다.
- **여러 속성을 한꺼번에 애니메이션할 때 유용**합니다.

`animate*AsState`를 여러 개 사용해도 다중 속성을 애니메이션할 수 있지만, **`updateTransition`이 더 적합한 경우가 있습니다.**

### **1. 애니메이션 간 동기화가 필요할 때**

- `animate*AsState`를 여러 개 사용하면 **각 애니메이션이 독립적으로 동작**해서 미묘한 타이밍 차이가 발생할 수 있음.
- `updateTransition`을 사용하면 **모든 속성이 같은 타이밍에 동기화되어 변화**함.

### **2. 공통 애니메이션 스펙을 적용할 때**

- `animate*AsState`를 여러 개 사용하면 각 애니메이션마다 **animationSpec을 개별로 설정**해야 함.
- `updateTransition`은 **공통 스펙을 적용할 수 있어 코드가 더 깔끔**해짐.

### **3. 애니메이션 상태 추적 및 조합이 필요할 때**

- `updateTransition`을 사용하면 **각 상태가 진행 중인지 확인**하고, `MutableTransitionState`를 활용하여 상태 기반 UI를 만들 수 있음.
- 예제코드

```kotlin
val transition = updateTransition(isExpanded, label = "Box Transition")

val size by transition.animateDp(label = "Size Animation") {
    if (it) 200.dp else 100.dp
}

val color by transition.animateColor(label = "Color Animation") {
    if (it) Color.Red else Color.Blue
}

// 애니메이션 진행 여부 확인
val isAnimating by transition.isRunning.collectAsState(initial = false)

Column {
    Box(
        modifier = Modifier
            .size(size)
            .background(color)
            .clickable { isExpanded = !isExpanded }
    )

    if (isAnimating) {
        Text("애니메이션 진행 중...", color = Color.White)
    }
}
```

![image.png](attachment:9f1028be-0cb7-4e85-b71a-7a55e580bd86:image.png)

정리해보면 단순한 애니메이션에는animate*AsState가 편리하지만, **복잡한 애니메이션을 만들 때는 `updateTransition`이 훨씬 강력**합니다

# `rememberInfiniteTransition` (무한 반복 애니메이션)

```kotlin
@Composable
fun InfiniteAnimation() {
    val infiniteTransition = rememberInfiniteTransition(label = "infinite transition")

    val size by infiniteTransition.animateFloat(
        initialValue = 50f,
        targetValue = 100f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000), // 1초 동안
            repeatMode = RepeatMode.Reverse // 커졌다 작아졌다 반복
        ),
        label = "size animation"
    )

    Box(
        modifier = Modifier
            .size(size.dp)
            .background(Color.Magenta)
    )
}

```

animationSpec에는 InfiniteRepeatable<T> 객체가 들어가는데 T 타입은 animate* 의 * 에 의해 정해집니다.

```kotlin
class InfiniteRepeatableSpec<T>(
    val animation: DurationBasedAnimationSpec<T>,
    val repeatMode: RepeatMode = RepeatMode.Restart,
    val initialStartOffset: StartOffset = StartOffset(0)
) : AnimationSpec<T> {

    @Deprecated(
        level = DeprecationLevel.HIDDEN,
        message = "This constructor has been deprecated"
    )
    constructor(
        animation: DurationBasedAnimationSpec<T>,
        repeatMode: RepeatMode = RepeatMode.Restart
    ) : this(animation, repeatMode, StartOffset(0))

    override fun <V : AnimationVector> vectorize(
        converter: TwoWayConverter<T, V>
    ): VectorizedAnimationSpec<V> {
        return VectorizedInfiniteRepeatableSpec(
            animation.vectorize(converter), repeatMode, initialStartOffset
        )
    }

    override fun equals(other: Any?): Boolean =
        if (other is InfiniteRepeatableSpec<*>) {
            other.animation == this.animation && other.repeatMode == this.repeatMode &&
                other.initialStartOffset == this.initialStartOffset
        } else {
            false
        }

    override fun hashCode(): Int {
        return (animation.hashCode() * 31 + repeatMode.hashCode()) * 31 +
            initialStartOffset.hashCode()
    }
}
```

중요한건 animation 파라미터로 들어오는 animationSpec 인데 대부분의 Animation API에서 개발자가 선택적 `AnimationSpec` 매개변수로 애니메이션 사양을 맞춤설정할 수 있습니다.

### **`spring`**

시작 값과 끝 값 사이에 물리학 기반 애니메이션을 만들며 두 매개변수 `dampingRatio` 및 `stiffness`를 사용

- `dampingRatio`는 스프링의 탄성을 정의

https://developer.android.com/static/develop/ui/compose/images/animations/damping_ratio.mp4?hl=ko

- `stiffness`는 스프링이 종료 값으로 이동하는 속도를 정의

https://developer.android.com/static/develop/ui/compose/images/animations/stiffness_annotated.mp4?hl=ko

### `tween`

[easing 곡선](https://www.google.com/search?sca_esv=fcc8a1ce826a5642&rlz=1C1OKWM_koKR945KR945&sxsrf=AHTn8zpRgsZQECRMVp_LzKlnZ-GVS5sByQ:1738844295031&q=easing&udm=2&fbs=ABzOT_CZsxZeNKUEEOfuRMhc2yCI6hbTw9MNVwGCzBkHjFwaKzX-IKePrWKWeOe-jIQvpgxNos-rYkZjCE31ffi49IJ7z7AaMexQ9JBDtqWJVNNzmIZaGx-Jza7pNgEW8kJP6Ud1aqnwMMTZ8pzMMTOri2rS2seMNnZEX1b_TRmMHFnRmQwC6iKeHm9qpOI0ROuy88DcGsIA&sa=X&ved=2ahUKEwiWspnkg6-LAxUKcPUHHc02B1cQtKgLegQIDhAB&biw=1920&bih=953&dpr=1#vhid=nckviGlKd6WtrM&vssid=mosaic)을 사용하여 지정된 `durationMillis` 동안 시작 값과 끝 값 간에 애니메이션을 처리합니다. `tween`는 두 값 *사이*를 이동하므로 '사이'의 줄임말
지정한 시간(duration) 동안 선형 또는 가속/감속 곡선을 따라 값을 변경하는 방식

```kotlin
val scale by animateFloatAsState(
    targetValue = if (animate) 2f else 1f,
    animationSpec = tween(
        durationMillis = 1000,  // 애니메이션 지속 시간 1초
        delayMillis = 200,      // 200ms 후 시작
        easing = LinearEasing   // 선형 속도 변화
    ), label = "scaleAnimation"
)
```

**`easing` 옵션**

`tween`의 `easing`을 조정하면 다양한 속도 변화를 적용할 수 있습니다.

- `LinearEasing` → 일정한 속도
- `FastOutSlowInEasing` (기본값) → 빠르게 시작하고 천천히 끝남
- `EaseInEasing` → 천천히 시작
- `EaseOutEasing` → 천천히 끝남
- `EaseInOutEasing` → 천천히 시작하고 천천히 끝남

### `keyframes`

애니메이션 기간에 여러 타임스탬프에서 지정된 스냅샷 값을 기반으로 애니메이션을 처리합니다. 언제나 애니메이션 값은 두 키프레임 값 사이에 보간됩니다.

```kotlin
val value by animateFloatAsState(
    targetValue = 1f,
    animationSpec = keyframes {
        durationMillis = 375
        0.0f at 0 using LinearOutSlowInEasing // for 0-15 ms
        0.2f at 15 using FastOutLinearInEasing // for 15-75 ms
        0.4f at 75 // ms
        0.4f at 225 // ms
    },
    label = "keyframe"
)
```

### `keyframesWithSplines`

값 간에 전환할 때 부드러운 곡선을 따르는 애니메이션을 만들려면 `keyframes` 애니메이션 사양 대신 `keyframesWithSplines`를 사용하면 됩니다.
**`keyframes`와 유사하지만, 각 키프레임 사이의 속도를 제어하기 위해 `Spline`을 사용할 수 있는** `AnimationSpec`입니다. `Spline`을 사용하면 애니메이션의 가속도나 감속도를 세밀하게 조정할 수 있어, 더 자연스럽고 부드러운 애니메이션을 구현할 수 있습니다.

```kotlin
val offset by animateOffsetAsState(
    targetValue = Offset(300f, 300f),
    animationSpec = keyframesWithSpline {
        durationMillis = 6000
        Offset(0f, 0f) at 0
        Offset(150f, 200f) atFraction 0.5f
        Offset(0f, 100f) atFraction 0.7f
    }
)
```
