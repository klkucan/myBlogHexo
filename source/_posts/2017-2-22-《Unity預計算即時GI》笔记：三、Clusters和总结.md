---
layout: post
title:  "《Unity預計算即時GI》笔记：三、Clusters和总结"
date:   2017-2-22 22:47:53 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 游戏开发
tags:
- Unity預計算即時GI
---



## 说明

- 这篇文章是对[《Unity預計算即時GI》](http://unitytaiwan.blogspot.tw/2016/12/unity-gi-1.html)这个系列文章的笔记。

## Clusters

>叢集，透過修改叢集(Clusters)也是一個降低Unity預計算流程所需要執行的工作數量的好方法。降低叢集數量也能提高執行時的效能。

>當採用PRGI來計算場景光照時，Unity會簡化產生一個立體像素化結構的計算，這些立體像素(Voxel)叫做叢集。叢集實際上是反映到場景靜態幾何表面用於照明的表面，叢集用一種層級關聯的結構來儲存，用來預計算Unity的全域光照漫反射所需要的複雜運算。雖然叢集和光照圖很像，但兩者用途是各自獨立的。

- 通过设置CPU Usage即可。

<!--more-->

## 微調光照參數

#### 创建
>要建立一個Lightmap Parameters資源，先找到Project視窗,
從Create下拉選單建立(Create > Lightmap Parameters)

>我們也可以在Project介面裡按右鍵選(Asset > Create > Lightmap Parameters) 來建立。

#### 使用

>從Hierarchy介面選擇你要指定變數集的物件，物件必須是帶有Mesh Renderer元件的靜態物件。

>開啟Lighting介面(Window > Lighting)並選擇Object頁籤

>從Advance Parameters下拉選單指定你的變數集給物件，右邊的"Edit"按鈕是開啟編輯光照變數的捷徑。

### 光照參數集說明

#### Resolution
>解析度的值確訂了物件採用的光照貼圖解析，這個值會和Lighting介面裡的解析度做加乘。比如說，如果場景解析度設為2，這裡的解析度設為0.5，那所有帶有這個參數集的物件都會採用1texel/unit來計算光照貼圖。

- 这个`Lighting介面裡的解析度`指的是[Scene-wide Realtime Resolution specified in the Scene tab of the Lighting window](https://unity3d.com/cn/learn/tutorials/topics/graphics/fine-tuning-lightmap-parameters?playlist=17102)，翻译的时候没说清楚啊。

#### Irradiance Budget

![image](https://unity3d.com/sites/default/files/dappling.png)
>解析度值很大光照貼圖所產生的影子斑點可以把Irradiance Budget這個參數調高來獲得緩解。

>注意，當解析度值很大時，在較低解析度下可能會產生奇怪的陰影，這些陰影在最終的光照貼圖裡可能看起來像是斑點或髒汙。如果有這種情況可以試著把Irradiance Budget參數提高來獲得改善。

#### Cluster Resolution

> 叢集解析度用來決定1個像素裡能有多少叢集數量。假如這個值設為1，代表光照圖裡面每個像素都都會有一個叢集，0.5代表一個像素會有2個叢集，換句話說叢集會是光照圖的兩倍大。 

>  imagine our Scene’s global Realtime Resolution was set to 1. We create a cube with a size of 1x1x1 units, and then assign a Lightmap Parameters asset to this object. If our Lightmap Parameters asset specified a Resolution of 1 and a Cluster Resolution of 1, we would have 1 Cluster per side of the cube. If we then increased our Resolution to 2, the result would be 2x(1x1) Clusters per side of the cube, giving 4 Clusters.

>將光照貼圖解析和叢集解析度保持指定比例，這樣我們可以和場景整體的解析度建立一個相對關係。我們可以把Lighting介面裡面的解析度定義為高解析度作為整體設定，然後針對個別物
件微調各自的光照參數集。

- 说白了，数值越大单位像素上cluster越多，与计算时间越长。

#### Irradiance Budget

>我們之前說明過光照計算是如何用叢集來計算靜態物件的預計算光照，在預計算的過程裡，叢集之間的關係被建立起來，好讓光線得以在叢集網內快速傳遞。

>在本質上，光照貼圖像素值的算法是基於叢集從該像素的位置對場景的一個檢視所計算得來，這會讓我們可以很快計算叢集之間的光照反射最後產生一個全域光照，這些叢集就能在畫面渲染完之前給予適當的樣本數。

>Irradiance Budget(輻照度範圍)用來制訂當叢集採樣時每個光照貼圖像素所使用到的記憶體量，**這會決定照明結果的精度，數值太低代表每個貼圖像素在記錄時使用較少記憶體同時提升CPU效能，代價就是會失真，數值越低光照結果會越模糊。反過來看，數值拉高GI會更準確，但記憶體和CPU的消耗都會提升。**

- 越低效率越高，适合大精细的模型，很大、模糊或者遥远的模型。

#### Irradiance Quality

>當計算PRGI時，每個光照貼圖像素會開始對場景投出射線，然後將可視資料報告給附近的叢集，然後貼圖像素就會得到每個叢集的百分比數值，這個值用來定義光照貼圖裡每個像素從叢集所分到的可視數據，**而一欄設定就是用來設定每個像素能對場景投射多少射線。**

>如果場景裡的物件和周圍物件光照不合的情況下可以斟酌加大這個值，有時該暗的時候光照結果卻意想不到的亮，有可能是因為投射到場景的射線不足或遮擋到，導致漏算叢集資料。同樣該亮的的放如果射線沒有檢查到，可能會造成過暗。

>提高射線的投射量就能解決類似的問題，代價就是增加預計算的時間，要優化這個時間，我們應該找出最適合的值來達到我們理想的照明效果。請注意，這個值不會影響到Runtime時的效能。

-还是越大越耗性能。

#### Backface Tolerance

> 當射線從光照貼圖像素投射出，從場景叢集蒐集光線時有時會打到幾何的背面，當計算全域光照時我們只需要關心投射到物體表面的光照，從背面來的光照資源通常都會忽略掉，這些從背面來的光照資料會破壞光照結果，因此調整這個值能防止這類情況發生。
![image](https://unity3d.com/sites/default/files/backfacetolerance.png)
>這裡的地板上的陰影就是Unity在計算期間從物件無效的背面所創造的，增加Backface Tolerance能改善這個問題。

> Backface Tolerance必須指定從前方光源來的百分比，好讓正面的像素被判定為有效。假如一個貼圖像素沒通過測試，Unity會採用鄰近的像素值嘗試算得正確光照資料。

> 調整這個值並不會影響PRGI運算效能，也不會對預計算時間長度有太大影響。反而是蠻適合在調整Irradiance Budget都無法解決的場景貼圖太亮或太暗問題時，Backface tolerance會是一個不錯的除錯工具。

- 調整這個值並不會影響PRGI運算效能，也不會對預計算時間長度有太大影響。2333333333 

## 总结

>學習如何評估專案場景並決定適合的光照解析度
了解光照圖，PRGI過程中最耗效能的元素之一，並學習如何降低它的數量。

- 核心，需掌握。

>學習如何幫小物件設定光照探針。

- 蛋疼的玩意。

>學習如何調整Unity的預計算參數，讓拆UV過程可以減少光照圖的數量。
了解甚麼是叢集，如何使用與它對全域光照的影響。

- 核心，需掌握。

>學習如何微調影響場景物件的光照貼圖變數，在不失真的情況下還能提高預計算效能。

- 还是比较有用的，主要还是通过影响cluster。

- 个人总结：这个系列的文章本质上是讲的如何减少光照图。通过改变分辨率、优化UV展开的结果等手段来实现。不过这篇文章如果单纯的看有点单薄，建议结合[《Unity 5 中的全局光照技术详解！》](http://mp.weixin.qq.com/s/uYX4T-fTgWxz_fWr6G4dPA)这篇文章看。回头看完在写笔记。

## 这个系列文章中提到的一些有意思的点

- PRGI只會呈現場景裡的漫反射(diffuse)和間接照明

- Unity的拆解演算法會嘗試把不同Shell做調整將UV邊緣拼接在一起來簡化UV貼圖

- 在某些情況下，網格匯入器可能會拆開幾何圖形。例如，如果有個網格有非常多的三角面，Unity可以為了效能把它分割成幾個獨立的子網格。通常這麼做是為了符合特定硬體需求，例如為了減少每個Draw Call所需要呼叫的三角面數量。分割通常會發生在相鄰的網格面之間法向角度有大變化的區域，比如銳角邊(hard edges)。這樣的拆分網格方式會在模型導入流程時執行，在這個過程中，UV Shell也可能會被拆分開來放到不同的光照圖，造成額外的光照圖消耗。

- 當計算PRGI時，每個光照貼圖像素會開始對場景投出射線，然後將可視資料報告給附近的叢集，然後貼圖像素就會得到每個叢集的百分比數值，這個值用來定義光照貼圖裡每個像素從叢集所分到的可視數據

- 當計算PRGI時，每個光照貼圖像素會開始對場景投出射線，然後將可視資料報告給附近的叢集，然後貼圖像素就會得到每個叢集的百分比數值，這個值用來定義光照貼圖裡每個像素從叢集所分到的可視數據。



 