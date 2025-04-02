---
layout: post
title:  "SwiftUI Picker 記憶體用量問題整理"
date:   2025-04-02 16:00:00 +0800
categories: swift-ui
tags: swift swift-ui picker snippet
---

近日，為實作選取語言功能，發現目前 SwiftUI 的 Picker 在記憶體使用方面，有極大的問題存在。

如下，實作一簡易App，只有單一元件用來選取語言，不同風格的 Picker 記憶體耗用如下:

- iOS

|style| init | expanded/scroll | collapsed | comment|
|---|---|---|---|---|
|automatic| | | | 預設值, 依平台及所在視圖等條件決定呈現方式|
|menu|~114 MB|~107 MB|~102 MB||
|inline|~65 MB|~53 MB|  | 配合 Form 使用 |
|navigationLink|~40 MB|~45 MB|~38 MB| 通常配合 NavigationStack 使用 |
|palette|~22 MB|~139 MB|~139 MB| 通常配合 Menu 使用 |
|segmented|~63 MB|~64 MB|  ||
|wheel|~43 MB|~43 MB|  ||

- macOS

|style| init | expanded/scroll | collapsed | comment|
|---|---|---|---|---|
|automatic| | | | 預設值, 依平台及所在視圖等條件決定呈現方式|
|menu|~108 MB|~120 MB|~120 MB|
|inline|~225 MB|~225 MB| | 直接使用時，呈現同 radioGroup, 搭配 Menu 時同 menu |
|radioGroup|~221 MB|~221 MB| | |
|palette|~22 MB|~139 MB|~139 MB|不適合文字資料，每次展開關閉都會增加記憶體|
|segmented|~310 MB|~310 MB| ~310 MB |||

## 分析

- 依據記憶體情況分析，menu, inline, segmented, wheel 等 style, Picker在初始化時，有可能是直接生成所有選項。
- palette 搭配 Menu 的情況下，可能是使用延後生成(lazy)策略，但收合Menu時，沒有釋放選項列表。
- macOS的記憶體用量，明顯比iOS多上許多。
- 查看memory graph，會發現如下情況:
    - macOS上 segmented 使用到 AppKit 物件約 ~35000，SwiftUI物件約 ~8600
    - iOS上 segmented 只使用SwiftUI物件約 ~12000
    - menu 情況下則各平台耗用 SwiftUI物件均在 ~400000 上下，明顯不合理。
        - 其中超過10000筆以上的資料，看來多半與外觀的 style/configuration等相關.
        - 考慮到測試資料約660筆左右，推測在實作上，swiftUI 可能多次複製同樣的資料
- 單純使用List + Text呈現資料時，耗用記憶體約 3~5 MB左右
- 使用 List + Button 呈現資料，耗用的記憶體也在 10~20 MB 之間

## 解法
- 需要由大量資料中，選取單一選項時，應避免使用Picker及Menu
- Picker的問題在於大量佔用記憶體，同時使用多個Picker會讓問題更嚴重。
- Menu必須搭配 Button 使用，雖然一開始不會佔用太多記憶體，但在展開一次後，同樣會一直佔用記憶體。同時，Menu內部實作看來與Picker有類似的問題，同樣數量的Button，配合Menu使用時，耗用的記憶體約為搭配List使用的2倍左右。
- 目前可行的解法 -- 自行實作類似元件，簡易範例如下:

```swift
struct IOSMenuPicker<SELECTION: Hashable, Content: View>: View {
    let title:String
    let content: () -> Content
    @Binding var selection: SELECTION
    @State private var isPicking = false
    
    var body: some View {
        Button {
            isPicking = true
        } label: {
            HStack {
                if !title.isEmpty {
                    Text("\(title)")
                }
                VStack {
                    Group {
                        Image(systemName:"control")
                        Image(systemName: "control").rotationEffect(.degrees(180.0))
                    }.font(.system(size: 10, weight: .bold))
                }
            }
        }.buttonStyle(.borderless).popover(isPresented: $isPicking) {
            ScrollViewReader { scroller in
                ScrollView {
                    LazyVStack {
                        MenuPickerBody(selection: $selection,
                                       isPicking: $isPicking,
                                       content: content)
                    }.padding(.vertical, 10)
                }.onAppear {
                    scroller.scrollTo(selection, anchor: .center)
                }
            }.frame(minWidth: 300).presentationCompactAdaptation(.popover)
        }
    }
    
    init(_ title: String, selection: Binding<SELECTION>, @ViewBuilder content: @escaping () -> Content) {
        self.title = title
        self._selection = selection
        self.content = content
    }
}

```
在iOS上實測，以上程式碼耗用記憶體初始 < 1 MB，展開後約增加 2~3 MB。
要注意的部分在於，雖然記憶體用量相較Picker及Menu少很多，但每次開閤Menu，還是會有記憶體些微增加的情況。


更進一步的改進，考慮到可供選取的資料較多，應可提供搜尋功能輔助完整列表。但搜尋功能多半與資料特性相關，要設計成泛用元件會有一定難度，在此不另做說明。

最後，附上詳細[範例程式](https://github.com/chenhaiteng/PickerMemoryDemo)。

