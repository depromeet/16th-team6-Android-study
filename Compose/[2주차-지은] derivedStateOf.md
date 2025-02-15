### 💡들어가기

- 해당 주제에 대한 선정이유 (순수하게 궁금해서 이런것도 OK)
    - 스터디 내용 & 클론 코딩 중에 몇번 등장하면서 궁금하기도 했고, recomposition을 방지하기 위한 방법으로 고민해보면 좋을 것 같아서 선정하였습니다
- 참고자료 레퍼런스
    - https://medium.com/androiddevelopers/jetpack-compose-when-should-i-use-derivedstateof-63ce7954c11b
    - https://developer.android.com/develop/ui/compose/side-effects#derivedstateof
  
<br></br>

## 🤓 derivedStateOf란?

- **기존 상태에서 파생된 새로운 상태를 계산**하고, 그 값이 변경되었을 때만 다시 계산되도록 하는 것
- ex) 두 개의 상태값이 있을 때 이 값들로부터 파생된 값을 계산하는데, 상태가 바뀔 때만 그 계산을 다시 하고 싶을 때 사용

<br></br>

## 😮 구체적으로 어떤 경우에 사용할까?

https://github.com/user-attachments/assets/bd56ee9c-ffc3-4edc-a395-631ab51aaf3f

사용자 이름이 작성되는 경우 버튼을 활성화시킴

→ ‘u’ 입력 시 버튼이 활성화 됨 (sumbitEnabled → true)

<br></br>

https://github.com/user-attachments/assets/b7032be8-92f7-4592-bd99-7b91552fc595

이후 남은 사용자 이름을 마저 입력

→ 나머지 알파벳을 입력할 때에도 sumitEnabled가 호출되어 버튼은 이미 활성화 되었지만 버튼에 계속 state를 보내 불필요한 true값이 쌓임

<br></br>

https://github.com/user-attachments/assets/e6dae805-2fe2-4ed0-b8da-2dd971ad73b1

이때 `derivedStateOf`를 사용하여 사용자 이름 입력을 감시하면 불필요하게 버튼에 state를 보내지 않음

→ u 입력 시 버튼이 활성화 되며 true를 보내고 이후 알파벳 입력엔 불필요한 상태를 보내지 않고, 올바르지 않은 문자(#)가 들어오면 false를 보내 버튼이 비활성화됨

<br></br>

### 또 어떤 경우에 사용하나면 ..

![image](https://github.com/user-attachments/assets/ff9d45e5-d409-4f1d-b448-bd5b86082995)

- 스크롤이 경계값을 통과하는지 (scrollPosition > 0)
- 리스트의 항목이 임계값보다 큰지 (아이템 > 0)
- 위와 같은 유효성 검사 (username.isValid())

<br></br>

## 🔍 derivedStateOf 내부 들여다보기

![image](https://github.com/user-attachments/assets/1dd1719d-58f7-40fe-b516-273cb99f4199)

**`derivedStateOf` 함수**

- `derivedStateOf`는 `calculation`이라는 람다식을 받아들여 계산된 결과를 `State`로 반환
- 이 상태는 `State`가 의존하는 다른 상태가 변경될 때만 다시 계산됨
- `derivedStateOf`의 기본은 "파생된 상태를 계산하고 그 상태가 변경되지 않으면 다시 계산하지 않는다"는 것

<br></br>

![image](https://github.com/user-attachments/assets/ca49a20f-821b-4bb6-870c-ef3cd22ea256)

**`DerivedSnapshotState`**

- `DerivedSnapshotState`는 파생된 상태의 실제 구현체로, 상태 값(result)을 관리하고, 그 값이 유효한지 체크
- 파생된 상태는 **`dependencies`**(다른 상태 객체)에 의존하며, 이러한 의존성들이 변경되었을 때 상태를 다시 계산
- **⭐️ 스냅샷을 이용한 계산 최적화**:
    - 스냅샷이란?
        - Compose에서 상태(State)를 추적하고 관리하는 시스템
        - “현재 상태의 복사본”을 만들어서 변경 사항을 효율적으로 감지하고 반영하는 역할을 함
    - `DerivedSnapshotState`는 상태를 계산할 때, 현재의 **스냅샷**을 사용하여 이전에 계산된 값이 유효한지 확인합니다.
    - 이 과정에서 스냅샷 ID나 작성 횟수(`writeCount`)를 추적하여, 상태가 불필요하게 다시 계산되지 않도록 합니다.

<br></br>

## 🤔 remember VS derivedStateOf

### remember

- 목적
    
    Composable 함수가 재구성될 때 상태 값을 기억하고, 그 값이 다시 계산되지 않도록 유지
    
- 동작 방식
    
    remember는 Composable 함수가 처음 호출될 때 초기화된 값을 저장하고, 이후 리컴포지션에서는 저장된 값을 재사용함
    

### derivedStateOf

- 목적
    
    다른 상태 값이 변경될 때 파생된 상태를 자동으로 계산
    
- 동작 방식
    
    다른 상태를 참조하여 파생된 값을 계산하고, 그 값이 의존성이 변경될 때만 다시 계산
    
    **의존 상태가 변경되지 않으면, 파생된 상태는 다시 계산되지 않음**
    
<br></br>

### 즉, derivedStateOf는 **UI를 업데이트 하려는 것보다 state 또는 key가 더 많이 변경되는 경우에 유용 !**
