---
title: RecyclerView smoothScrollToPosition의 스크롤 위치가 정확하지 않을 때
categories:
  - Android
tags:
  - Android
  - RecyclerView
  - Data Binding
---

`RecyclerView`에서 `smoothScrollToPosition()`을 사용할 때 간헐적으로 스크롤 위치가 살짝 어긋나는 이슈가 발생했다.  

{% include video id="kevLdOEnbII" provider="youtube" %}

코드는 다음과 같다.  

```kotlin
class ScrollToTopAdapter : ListAdapter<String, ScrollToTopViewHolder>(diffCallback) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ScrollToTopViewHolder =
        ScrollToTopViewHolder.create(parent)

    override fun onBindViewHolder(holder: ScrollToTopViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

class ScrollToTopViewHolder(
    private val binding: ItemScrollToTopBinding,
) : RecyclerView.ViewHolder(binding.root) {

    fun bind(text: String) {
        binding.text = text
    }

    companion object {
        fun create(parent: ViewGroup): ScrollToTopViewHolder {
            val binding = ItemScrollToTopBinding.inflate(
                parent.context.layoutInflater,
                parent,
                false
            )
            return ScrollToTopViewHolder(binding)
        }
    }
}
```

사실 해당 이슈는 아래 두 가지 조건을 만족할 때만 발생할 수 있다.  
1. `ViewHolder`에서 Data Binding을 사용할 때
2. 리스트 아이템 간의 높이가 서로 다를 때

`smoothScrollToPosition()` 호출 시 스크롤 방향의 아이템들 사이즈를 측정하여 최종 스크롤 위치를 계산하는 것으로 보이는데, 데이터 바인딩을 사용 중인 경우 기본적으로 바인딩 시점이 다음 프레임으로 지연되는 것이 원인이다.  
아이템 사이즈 측정 시점에는 아직 바인딩이 진행되기 전이므로, 데이터에 따라 아이템의 높이가 달라지는 경우 정확한 높이를 측정할 수 없다.  
따라서 데이터 바인딩을 지연하지 않고 즉시 실행하는 `executePendingBindings()`를 `Adapter`의 `onBindViewHolder()` 시점에 호출하면 해결된다.

```kotlin
class ScrollToTopViewHolder(
    private val binding: ItemScrollToTopBinding,
) : RecyclerView.ViewHolder(binding.root) {

    fun bind(text: String) {
        binding.text = text
        binding.executePendingBindings()
    }
    
    // ...
}
```