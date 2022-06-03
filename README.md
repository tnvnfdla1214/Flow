# StateFlow란

https://kotlinworld.com/232

최근 코틀린 코루틴 라이브러리 (1.4.1)에서 Stable API로 배포된 StateFlow, SharedFlow가 무엇인지와 이들로 LiveData를 대체할 수 있는지를 알아보려고 합니다.
### 실 예시 프로젝트
[User_StateFlow](https://github.com/tnvnfdla1214/User_StateFlow)

### StateFlow
+ 기본적으로 상태 처리를 위한 새로운 기본 요소입니다.
+ StateFlow는 값이 업데이트 된 경우에만 반환하고 동일한 값을 반환하지 않습니다. 
+ Flow는 일반적으로 [cold stream](https://3edc.tistory.com/69)이지만, StateFlow는 [hot stream](https://3edc.tistory.com/69)입니다. 즉 일반 Flow는 마지막 값의 개념이 없고 collect 될 때만 활성화 되는 반면 StateFlow는 마지막 값의 개념이 있으며 생성하자마자 활성화됩니다.
+ StateFlow는 ConflatedBoradcastChannel을 대체하기 위해 설계되었습니다. StateFlow는 기본 콘셉트로 distinctUntilChanged 연산자의 기능을 가지고 있기 때문에, **업데이트될 때 같은 값이라면 필터 됩니다.**
+ Android에서 StateFlow는 식별 가능한 변경 가능 상태를 유지해야 하는 클래스에 적합합니다.

### SharedFlow
+ ShraredFlow는 StateFlow의 일반화로 생각할 수 있습니다.
+ StateFlow는 기본적으로 새 구독자가 있을 때 마지막으로 알려진 값을 내 보냅니다. SharedFlow를 사용하면 내보낼 이전 값 수를 구성할 수 있습니다.
+ 값의 버퍼가 가득 차면 어떤 일이 발생하는지 정의할 수 있습니다.
+ 이는 LiveData가 쉽게 수행할 수 없는 고급 사용 사례에 대해 뛰어난 유연성을 제공합니다.

### StateFlow vs LiveData
일반적으로 많은 개발자들이 안드로이드 라이프사이클을 위해 사용하는 LiveData를 StateFlow로 대체할 수 있다고 하는데요.

둘을 비교해 보게 되면 비슷한 점이 있습니다. 둘 다 식별 가능한 데이터 홀더 클래스이며, 앱 아키텍처에 사용할 때 비슷한 패턴을 따릅니다.

그럼 안드로이드에서 StateFlow를 사용하는 예제를 살펴보겠습니다.

 ```Kotlin
class MyViewModel(private val repository: Repository) : ViewModel() {
    private val userNameFlow = MutableStateFlow("")
    val userName: StateFlow<String> = userNameFlow
    
    init { viewModelScope.launch {
    userNameFlow.value = repository.fetchUserName()
    }
  }
}
```
LiveData를 사용할 때와 상당히 유사한 점들이 보입니다.

LiveData를 사용할 때처럼 backing property(MutableLiveData)를 만들어 관리해주는 방법과 유사한데요.

그러나 StateFlow와 LiveData는 다음과 같이 다르게 동작합니다.

1. StateFlow의 경우 초기 값이 필요하지만, LiveData의 경우는 필요하지 않습니다.
2. 뷰가 STOPPED 상태가 되면 LiveData.observe()는 소비자를 자동으로 등록 취소하는 반면, StateFlow 또는 다른 흐름에서 수집의 경우 자동으로 중지하지 않습니다.
![image](https://user-images.githubusercontent.com/48902047/149606228-ad485e61-3104-41f1-ab6c-41c09c2e8a9c.png)

MainActivity에서 위의 MainViewModel 구현하게 된다면 아래와 같습니다.

 ```Kotlin
class MyActivity : AppCompatActivity() {
      private val viewModel = getViewModel()
      
      override fun onCreate(savedInstanceState: Bundle?) {
        lifecycleScope.launchWhenStarted {// 2.
          viewModel.userName.collect { userName ->
            userNameLabel.text = userName
          }
        }
     }
}
```
collector를 가지고 값이 업데이트가 되는 것을 지켜봐야 하는데, StateFlow에는 라이프 사이클에 따라 자동으로 중지되는 기능이 없기 때문에 뷰에서 lifecycleScope를 이용하여 생명주기 인지를 시켜줘야 합니다. 

위의 2번의 자동으로 소비를 관리해주지 않는 특징은 오히려 불편해 보일 수 있습니다. 이 부분은 lifecycle-livedata-ktx의 확장 함수인 asLiveData()를 통해 해결해 줄 수 있습니다.

 ```Kotlin
class MyActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
      viewModel.userName.asLiveData().observe(viewLifecycleOwner) {
        userName -> userNameLabel.text = userName
      }
    }
}
```
지금까지 StateFlow의 특징과 LiveData와 차이점을 알아봤습니다. 이전 글에서 Flow와 LiveData의 차이점과 함께 사용시에 이점을 알아 봤는데, StateFlow가 LiveData를 대체한다면 Flow API로만 앱을 빌드 할 수 있을지에 궁금증이 생깁니다. 

그럼 정말 StateFlow가 LiveData를 대체할 수 있는 걸까요? 

위의 내용들로 뒷받침할 수 있다고 합니다.

+ LiveData는 Android Lifecycle에 따라 달라지기 때문에 Views / ViewModels 외부에서 사용하기에 적합하지 않습니다.
+ StateFlow는 LiveData와 같은 동작을 하지만, 더 많은 연산자(combind, zip, ect)와 더 좋은 성능을 가지고 있습니다.
+ StateFlow API는 LiveData와 거의 동일하며 SharedFlow는 고급 사용 사례를 위한 유연성을 제공합니다.
+ StateFlow는 LiveData (앱이 백그라운드로 이동하면 자동 구독 / 구독 취소)의 모든 이점을 얻을 수 있으므로 가치가 있습니다.













