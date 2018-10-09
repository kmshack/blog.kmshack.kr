---
title: 안드로이드에서 자연스러운 키보드 변경 처리하기
tags: 안드로이드
layout: post
comments: true
---

[Dank](https://play.google.com/apps/testing/me.saket.dank)라는 앱을 사용할때 많은 사용자들은 부드럽게 잘 작동한다는 것에 만족합니다. 앱은 화면변경시 적절한 모션처리를 잘하였지만, 앱이 좀더 부드럽게 느껴지는 이유는 키보드 변경처리를 잘하였기 때문입니다.  

소프트 키보드가 표시되면 안드로이드는 콘텐츠의 크기를 즉시조정합니다. (`windowSoftInputMode`를 설정하지 않은경우) 이로 인해 레이아웃에 부자연스러운 변화가 생깁니다. 안타깝게도 플랫폼이나 대다수의 앱이 이를 처리하지 않아 사용자들은 정상적으로 작동하는것으로 착각합니다.   


차이점을 명확히 알아보기 위해 Google Keep과 Dank 앱을 비교 해보겠습니다.  

![](/images/2018-10-09-smooth_keyboard/1.gif){:height="40%" width="40%"}
Google Keep  

![](/images/2018-10-09-smooth_keyboard/2.gif){:height="40%" width="40%"}
Dank  

<br>

비교해 보면 Dank앱이 키보드가 반응하면 콘텐츠를 위쪽으로 부드럽게 크기를 조정하며 스크롤하게 됩니다. 반면 Google Keep은 컨텐츠의 변동없이 키보드의 반응에 아무런 반응이 없어 보입니다.  

좀 더 자세한 비교를 원한다면 영상으로 확인해보세요.( [Google Keep](/images/2018-10-09-smooth_keyboard/keep_keyboard_720p.mp4), [Dank](/images/2018-10-09-smooth_keyboard/dank_keyboard_720p.mp4))  

<br>

눈치 빠른 분들은 Dank앱이 처리하는 작은 트릭을 발견하실 수 있습니다. 키보드가 표시되면 Activity의 전체 화면의 크기가 조정되지 않는다는 것을 알게되었습니다. Activity를 구성하는 View 계층의 최상단 레이아웃(`DecorView`)에 아무런 영향을 받지 않습니다. 실제로 크기가 저장되는 레이아웃은 ID가 할당된 콘텐츠 레이아웃(`android.R.id.content`)입니다. 이것은 `DecorView`가 차지하는 공간을 제어할 수 있다는 것을 의미합니다.

Activity를 구성하는 View Tree는 일반적으로 다음과 같습니다.  

**DecorView**  
```java
- LinearLayout
-- FrameLayout  <- android.R.id.content
--- LinearLayout
---- Activity content
```

![](/images/2018-10-09-smooth_keyboard/3.gif){:height="40%" width="40%"}  

리사이즈 되는 부분을 제어하기 위해 콘텐츠 레이아웃의 크기가 변경되는 것을 감지하는 유틸 클래스를 만들었습니다. View의 크기가 변경되면 컨텐츠의 전체 높이에서 변경된 높이만큼 변경되도록 애니메이션처리 합니다.  

<br>

```java
val decorView = activity.window.decorView

decorView.viewTreeObserver.addOnPreDrawListener { 
  val contentHeight = contentViewFrame.height
  val sizeChanged = contentHeight != previousHeight

  if (sizeChanged) {
    animateSizeChange(from = previousHeight, to = contentHeight)
  }

  previousHeight = contentHeight
}
```

![](/images/2018-10-09-smooth_keyboard/4.gif){:height="40%" width="40%"}



```java
fun animateSizeChange(from: Int, to: Int) {
  // Immediately snap back to the original size.
  contentView.setHeight(from)
  
  ObjectAnimator.ofInt(from, to)
    .addUpdateListener { 
      val h = it.animatedValue as Int
      contentView.setHeight(h) 
    }
    .start()
}
```

관련된 모든 코드는 [여기](https://github.com/saket/FluidKeyboardResize)를 통해 확인하세요.  

<br>

`decorView`의 컨텐츠 크기에 따라 애니메이션처리를 하여 높이를 변경하는 것에 대해서 손이 많이 가지만 앱의 동작에 큰 영향을 미쳤습니다. 이것은 완벽한 해결책은 아니며 다른 방법의 해결책도 있음을 이해하고 있습니다. 그러나 여러달 사용해왔으며 어떤 이슈도 발견하지 못했습니다.


<br>
<br>
참고: [https://saket.me/smoothly-reacting-to-keyboard/](https://saket.me/smoothly-reacting-to-keyboard/)