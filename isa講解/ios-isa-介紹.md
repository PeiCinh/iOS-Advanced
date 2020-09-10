---
title: "iOS ISA 介紹 "
tags: ""
---

1.  # `isa`是什麼?
    isa 是將地址與類別作結合

2.  # `isa` 在哪裡
    有如下圖片，在控制台輸出objc1的數據結構，排在第一位的就是`isa`的地址。
    ![alt text](https://github.com/PeiCinh/iOS-Advanced/blob/master/isa%E8%AC%9B%E8%A7%A3/img/isa1.png?raw=true "Logo 標題文字範例一")

為什麼呢？因為類別繼承自`NSObject`，`NSObject`在底層的實現是已結構`objc_object`，里面只有一個`isa`變數，那麼類別的首地址指向的第一塊就是isa所在位置。

![Image][isa3]

3.  # `isa` 類型
    照原理來說 `isa` 是指對象的類型，在下面打印地址一的值，但這邊出乎意料的是打印出來一堆數字。

![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa3.png?raw=true)

接著看原碼，發現`isa`是`Class` 。

![Image][isa3]

`Class`類型其實是`objc_class`。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa4.png?raw=true)

`objec_class` 繼承著 `objc_object` 那裡面就有一個 `isa`
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa5.png?raw=true)

4.  # `isa` 結構
    看上面那麼久還是不知道`isa`是甚麼，在之前 `alloc` 初始化中有看到一開始計算size ，然後開闢內存，最後 `initInstaceIsa`跟`initIsa`兩個方法，那我們就可以朝裡面查看`isa`到底是怎樣的結構。
    ![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa7.png?raw=true)

原來`isa`是個聯合體，那重點在 `ISA_BITFIELD`。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa6.png?raw=true)

 終於看到`ISA_BITFIELD` 這邊不同平台要關注不同 `ISA_BITFIELD` 參數。參數這邊前面是名稱，後面是占用位元，全部加起來是`64`bit，不管在`arm64`還是`x86`都是 `64`bit。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa8.png?raw=true)

| 参数名               |                                           作用                                           | 大小 |  所在位置 |
| ----------------- | :------------------------------------------------------------------------------------: | -: | :---: |
| nonpointer        |                是否對isa指針開啟指針優化0：純isa指針只包含類對像地址1：isa中包含了類對像地址、類信息、對象的引用計數等               |  1 |   0   |
| has_assoc         |                                    是否有關聯對象0：沒有 1：存在                                    |  1 |   1   |
| has_cxx_dtor      |                    該對像是否有C++或者Objc的析構器 如果有析構函數則需要做析構邏輯如果沒有則可以更快的釋放對象                   |  1 |   2   |
| shiftcls          |                        存儲類指針的值。開啟指針優化的情況下，在arm64架構中有 33 位用來存儲類指針                       | 33 |  3~35 |
| magic             |                               用於調試器判斷當前對像是真的對像還是沒有初始化的空間                               |  5 | 36~40 |
| weakly_referenced |                                    是否有弱引用 0：沒有 1：存在                                    |  1 |   41  |
| deallocating      |                                    是否正在釋放內存 0：不是 1：是                                   |  1 |   42  |
| has_sidetable_rc  |                         是否需要用到外掛引用計數，當對象引用技術大於 10 則需要藉用該變量存儲進位                         |  1 |   43  |
| extra_rc          | 該對象的引用計數值，實際上是引用計數值減 1。如果對象的引用計數為10，那麼 extra_rc 為 9。如果引用計數大於 10 則需要使用 has_sidetable_rc | 19 | 44~63 |

打印去 `isa` 出來，`cls` 這邊已顯示當前類別，那另外`shiftcls`的這串數字到底是什麼，不應該是當前對象的指標嗎?
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa9.png?raw=true)

5.  # `isa` 獲取類別
    那我們看下原本，原來`shiftcls`向右移三位
    ![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa10.png?raw=true)

那我們將`shiftcls`向左移三位，就能還原對象，這些終於找到對象的類別。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa11.png?raw=true)

6.  # 如何從對象獲取類別?

平常我們都從對象直接調取類別，那現在我們從原碼開始看他怎麼實現的。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa12.png?raw=true)

這時發現取得對象類別時會調用 `getIsa` 。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa13.png?raw=true)

接著 `isTaggedPointer` 是否是Tagged Pointer類型，這邊不是直接返回 `ISA()`。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa14.png?raw=true)

這邊由於不是 `SUPPORT_INDEXED_ISA`，最後的一行就是`isa.bits`和`ISA_MASK`做運算。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa15.png?raw=true)

在前面我就有看到`ISA_MASK`，這邊在SHOW一下。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa8.png?raw=true)

那我們驗證一下上面的公式 `bit` & `MASK`，這將剛剛打印的`bit` & `MASK`做運算，結果得出當前對象類別。
![Image](https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa16.png?raw=true)

[isa3]: https://github.com/PeiCinh/iOS-Advanced/blob/feature/isa-explain/isa%E8%AC%9B%E8%A7%A3/img/isa2.png?raw=true
