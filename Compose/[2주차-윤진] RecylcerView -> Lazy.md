### 💡 들어가기

- 해당 주제에 대한 선정이유 (순수하게 궁금해서 이런것도 OK)
    - 실습을 진행하다보니 문득 RecyclerView가 없는데 목록이 이렇게 쉽게 구현될 수가 있다는게 신기해서 ~ !!
- 참고자료 레퍼런스
    - https://developer.android.com/develop/ui/compose/migrate/migration-scenarios/recycler-view?hl=ko
    - https://velog.io/@kijrary/Android-Jetpack-Compose%EC%97%90-%EB%8C%80%ED%95%B4-RecyclerView-Layout
    - https://medium.com/hongbeomi-dev/recyclerview-%EC%95%84%EC%9D%B4%ED%85%9C%EC%9C%BC%EB%A1%9C-composeview%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-2dd7ae69c6bb
    - https://m1nzi.tistory.com/14
- 나의 생각이 포함된다면 그러한 생각의 원인과 결론이 어떻게 됐는지
- 등에 대한 내용이 포함되게 작성해주세요
</aside>

## RecyclerView → Compose 이전 단계

<aside>

1. **RecyclerView의 각 ViewHolder를 ComposeView로 마이그레이션**
2. **RecylcerView를 Compose의 리스트 컴포넌트로 마이그레이션**
    1. **LinearLayoutManager → LazyColumn / LazyRow**
    2. **GridLayoutManager → LazyVerticalGrid / LazyHorizontalGrid**
    3. **StaggeredGridLayoutManager → LazyVerticalStaggeredGrid / LazyHorizontalStaggeredGrid**
</aside>

<aside>

이 때 주의할 점은 !! ViewCompositionStrategy를 주의해서 UI를 구현해야 한다 !!

</aside>



# ❗ComposeView의 ViewCompositionStrategy

## ViewCompositionStrategy

- composition을 dispose 해야하는 시기를 설정하는 것
- **compose 메모리가 언제 해제될 지 전략을 정해주는 것인데, 적절한 시기에 해제하지 않으면 메모리 누수가 발생할 수 있기 때문에 중요하다!**

### Composition disposing

- composition이 dispose되면, 리소스는 정리되고 state는 Compose에 의해 더이상 트래킹되지 않는다.
- 적용된 특정 strategy는 composition이 자동으로 dispose되는 시기를 결정한다.
- setViewCompositionStrategy를 통해 다른 strategy를 적용할 수도 있다.
    
    ```kotlin
    val composeView = ComposeView(context = context)
    composeView.setViewCompositionStrategy(
    	ViewCompositionStrategy.DisposeOnViewTreeLifeCycleDestroyed
    )
    ```
    

## 종류

### DisposeOnDetachedFromWindow (default)

- ComposeView가 Window에서 detach됐을 때 composition이 dispose된다.
    - View가 Screen에서 벗어나서 사용자에게 더이상 표시가 되지 않을 때
        - `ViewGroup.removeView` 를 통해 View가 제거됐을 때
        - Activity destroy

### DisposeOnDetachedFromWindowOrReleasedFromPool

- ComposeView가 RecyclerView와 같은 pooling container 내에서 사용될 때
- UI가 스크롤로 재활용됨에 따라 view element가 window에 attach, re-attach 될 때 dispose 된다.
- **RecyclerView 같은 경우 스크롤 중에 각 아이템에 대하여 Composition을 자주 dispose + recreate하면 버벅거리고 스크롤 성능이 저하될 수 있는데 이를 방지할 수 있다 !!**

### DisposeOnLifecycleDestroyed

- 설정된 lifecycle이 destroy됐을 때 dispose된다.
- ComposeView : lifecycle이 일대일 관계일 때 적합
- Fragment 환경의 ComposeView에서 사용하기 적합함
    - View는 detach되었으나, Fragment가 destroy되지 않을 수 있음

```kotlin
class MyFragment : Fragment() {
      // …
      override fun onCreateView(
          inflater: LayoutInflater,
          container: ViewGroup?,
          savedInstanceState: Bundle?
      ) = ComposeView(requireContext()).apply {
          **setViewCompositionStrategy(
              ViewCompositionStrategy.DisposeOnLifecycleDestroyed(
                  lifecycle = this@MyFragment.lifecycle // <== Lifecycle 지정
              )
          )**
          // …
      }
  }
```

### DisposeOnViewTreeLifecycleDestroyed

- View가 attach된 다음 현재의 window에서 ViewTreeLifecycleOwner가 destroy될 때 dispose된다.
- Fragment View와 같이 ViewTreeLifecycleOwner와 일대일 관계일 때 적합함
- Lifecycle을 모를 경우 사용한다.

<aside>

> 만약 Fragment 내에 RecyclerView가 있고, RecyclerView의 item에 ComposeView가 있다면?
     **DisposeOnDetachedFromWindowOrReleasedFromPool** 사용 !
        - ComposeView는 RecyclerView의 item이기 때문이다.
        - 그렇지 않은 경우, DisposeOnLifecycleDestroyed 사용
</aside>



# Migration

## 1. ViewHolder를 ComposeView로 migration

### lift_gift_item.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <variable
            name="vm"
            type="com.dangjang.android.presentation.mypage.MypageViewModel" />

        <variable
            name="giftList"
            type="com.dangjang.android.domain.model.ProductVO" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:id="@+id/gift_list_cl"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="8dp"
            android:layout_marginBottom="8dp"
            android:background="@drawable/background_white_gradient"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent">

            <TextView
                android:id="@+id/textView5"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginStart="16dp"
                android:layout_marginTop="16dp"
                android:text="@{giftList.title}"
                android:textColor="@color/black"
                android:textSize="16sp"
                android:textStyle="bold"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toTopOf="parent" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginStart="16dp"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="16dp"
                addPointUnit="@{giftList.price}"
                android:textColor="@color/black"
                android:textSize="14sp"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@+id/textView5" />

            <ImageView
                android:id="@+id/gift_list_check_iv"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="32dp"
                android:src="@drawable/ic_check_green"
                android:visibility="gone"
                app:layout_constraintBottom_toBottomOf="parent"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintTop_toTopOf="parent" />
        </androidx.constraintlayout.widget.ConstraintLayout>

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```

### GiftItemView (ComposeView로 전환)

```kotlin
@Composable
fun GiftItemView(viewModel: MypageViewModel, giftList: ProductVO, isChecked: Boolean) {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .padding(vertical = 8.dp)
            .background(
                painterResource(id = R.drawable.background_white_gradient),
                shape = RoundedCornerShape(8.dp)
            )
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
        ) {
            Text(
                text = giftList.title,
                fontSize = 16.sp,
                fontWeight = FontWeight.Bold,
                color = Color.Black
            )

            Spacer(modifier = Modifier.height(8.dp))

            Text(
                text = "${giftList.price} 원",
                fontSize = 14.sp,
                color = Color.Black
            )
        }

        if (isChecked) {
            Image(
                painter = painterResource(id = R.drawable.ic_check_green),
                contentDescription = "Checked",
                contentScale = ContentScale.Fit,
                modifier = Modifier
                    .align(Alignment.TopEnd)
                    .padding(end = 32.dp)
            )
        }
    }
}

```

### 기존 ViewHolder

```kotlin
    inner class ViewHolder(val binding: ItemGiftListBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(giftListItem: ProductVO) {
            binding.vm = viewModel
            binding.giftList = giftListItem
        }
    }
```

### 변경된 ViewHolder

```kotlin
		inner class ViewHolder(private val composeView: ComposeView) :
        RecyclerView.ViewHolder(composeView) {

        fun bind(giftListItem: ProductVO) {
            composeView.setContent {
                GiftItemView(viewModel, giftListItem, isChecked)
            }
        }
    }
```

<aside>

실제로 Compose로 migration을 할 때, 이렇게 ViewHolder를 없애기 전에 item list만 ComposeView로 전환하여 **RecylcerView + Compose 혼합 사용**도 많이 한다고 한다 !!

</aside>

### 기존 View에서 Jetpack Compose를 렌더링하는 방식

1. 기존 xml에 ComposeView 추가

```kotlin
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/composeView"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

1. 코드에서 동적으로 추가

```kotlin
val composeView = ComposeView(context).apply {
    **setContent** {
        GiftItemView(viewModel, giftListItem, isChecked)
    }
}
parentLayout.addView(composeView) // 기존 ViewGroup에 추가
```

## 2. **RecylcerView를 Compose의 리스트 컴포넌트로 마이그레이션**

### 기존 RecyclerView xml 코드

```xml
                <androidx.recyclerview.widget.RecyclerView
                    android:id="@+id/point_gift_rv"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:layout_marginHorizontal="10dp"
                    android:layout_marginTop="16dp"
                    android:layout_marginBottom="24dp"
                    android:orientation="vertical"
                    **app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"**
                    app:layout_constraintBottom_toTopOf="@+id/next_btn"
                    app:layout_constraintEnd_toEndOf="parent"
                    app:layout_constraintStart_toStartOf="parent"
                    app:layout_constraintTop_toBottomOf="@+id/point_gift_title_tv" />

```

### Compse LazyColumn으로 변경

- LinearLayoutManager 필요 없음

```kotlin
@Composable
fun GiftListScreen(viewModel: MypageViewModel, giftList: List<ProductVO>) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 10.dp, vertical = 16.dp)
    ) {
        **LazyColumn**(
            modifier = Modifier
                .fillMaxWidth()
                .padding(bottom = 24.dp)
        ) {
            items(giftList) { giftItem ->
                GiftItemView(viewModel, giftItem, isChecked = false)
                Spacer(modifier = Modifier.height(8.dp))
            }
        }
    }
}
```

→ 이렇게 다 migration 하게 된다면, **ViewHolder, Adapter가 삭제**되어 코드가 훨씬 간결해진다 !!

---

## **LazyColumn vs RecyclerView**

<aside>

RecyclerView는 `ViewHolder`를 이용하여 뷰를 재활용하는 반면, LazyColumn은 Compose의 리컴포지션(recomposition) 메커니즘을 통해 뷰 노드를 관리한다.

</aside>

- **LazyColumn**은 화면에 보이는 뷰만 생성하고, 스크롤 시 화면 밖으로 사라지는 뷰는 삭제, 새로 생기는 뷰는 새로 생성한다. 이 과정은 지능적 리컴포지션을 바탕으로 하며, 목록 전체가 아니라 특정 컴포넌트만 리컴포지션될 수 있게 한다.
- **RecyclerView**는 `ViewHolder`를 사용하여 뷰를 재사용한다. `ViewHolder`는 뷰 계층 구조를 보유하며 데이터를 바인딩한다. RecyclerView는 `ViewHolder`들을 풀에 저장하고 필요에 따라 재사용한다.

## **LazyColumn > RecyclerView**

### 1️⃣ 여러 ViewType 설정

**`RecyclerView`**

- ViewHolder 2종류 만들어야 함.

```kotlin
override fun getItemViewType(position: Int): Int {
    return if (position % 2 == 0) TYPE_ONE else TYPE_TWO
}

override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
    return if (viewType == TYPE_ONE) {
        ViewHolderTypeOne(LayoutInflater.from(parent.context).inflate(R.layout.item_one, parent, false))
    } else {
        ViewHolderTypeTwo(LayoutInflater.from(parent.context).inflate(R.layout.item_two, parent, false))
    }
}

```

**`LazyColumn`**

- if문으로 해결

```kotlin
LazyColumn {
    items(items) { item ->
        if (item.startsWith("A")) {
            Text(text = "Type A: $item", color = Color.Red)
        } else {
            Text(text = "Type B: $item", color = Color.Blue)
        }
    }
}
```

### 2️⃣ 무한 스크롤

**`RecyclerView`**

- addOnScrollListener()를 사용

```kotlin
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrolled(recyclerView: RecyclerView, dx: Int, dy: Int) {
        val layoutManager = recyclerView.layoutManager as LinearLayoutManager
        val lastVisibleItem = layoutManager.findLastVisibleItemPosition()
        if (lastVisibleItem == adapter.itemCount - 1) {
            loadMoreData()
        }
    }
})
```

**`LazyColumn`**

- itemsIndexed()의 마지막 항목 감지 기능을 활용하여 간단하게 구현

```kotlin
LazyColumn {
    itemsIndexed(items) { index, item ->
        Text(text = item)
        **if (index == items.lastIndex) {
            CircularProgressIndicator() // 로딩 UI 표시
        }**
    }
}
```

### 3️⃣ Header, Footer 추가

**`RecyclerView`**

- HEADER, FOOTER ViewType을 만들어 따로 처리

```kotlin
override fun getItemViewType(position: Int): Int {
    return when (position) {
        0 -> TYPE_HEADER
        items.size + 1 -> TYPE_FOOTER
        else -> TYPE_ITEM
    }
}
```

**`LazyColumn`**

- item()을 사용하여 쉽게 추가 가능

```kotlin
LazyColumn {
    item { Text(text = "Header", fontSize = 20.sp, fontWeight = FontWeight.Bold) }
    items(items) { item -> Text(text = item) }
    item { Text(text = "Footer", fontSize = 16.sp, color = Color.Gray) }
}
```

### 4️⃣ 아이템 ClickListener 처리

**`RecyclerView`**

- ViewHolder에서 setOnClickListener()를 사용

```kotlin
holder.itemView.setOnClickListener {
    onItemClickListener.invoke(itemList[holder.adapterPosition])
}
```

**`LazyColumn`**

- Modifier.clickable 사용

```kotlin
LazyColumn {
    items(items) { item ->
        Text(
            text = item,
            modifier = Modifier
                .fillMaxWidth()
                .**clickable** { println("Clicked on $item") }
                .padding(16.dp)
        )
    }
}
```

### 5️⃣ 리스트가 자주 변경되는 경우

**`RecyclerView`**

- notifyDataSetChanged(), notifyItemInserted(), notifyItemRemoved() 등을 호출해야한다.

**`LazyColumn`**

- UI 재구성을 자동으로 처리하므로 리스트가 자주 변경되는 경우 훨씬 편리하다.

## **LazyColumn < RecyclerView**

- 리스트 개수가 매우 많고, 빠른 스크롤이 필요할 때
    - RecyclerView는 Recycler Pool을 사용해 빠른 스크롤을 지원하고, setHasFixedSize(true), RecycledViewPool 등을 사용해 최적화가 가능하다.
- ViewType이 많거나, 애니메이션 최적화가 필요할 때
    - RecyclerView는 getItemViewType(), ItemAnimator, SpanSizeLookp, DiffUtil 같은 기능으로 성능 최적화 가능



# 결론

Compose 기반 프로젝트로 빠르게 개발하려면 LazyColumn 사용하자 !!

성능과 확장성을 더 고려하려면 RecyclerView 사용하자 !!