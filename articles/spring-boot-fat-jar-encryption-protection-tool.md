---
title: 做了一個 Spring Boot 的 Fat Jar 加密保護工具
date: 2025-11-03
tags: [java, spring-boot]
---

因為篇幅的原因，直接忽略所有採坑的過程進入重點。

功能的實現始於一項主管的需求：公司需要將專案部屬在客戶的主機上運行，但因為技術專利的緣故，老闆希望原始碼的部分內容可以進行加密處理，同時不影響上線後的系統功能，因此需要我們做出一個「可執行的、經加密後的 spring boot fat jar」給客戶運行。需求的難點主要有 2 個：第一個是相較於 spring boot 的系統底層，我對 spring boot fat jar 打包與實際運行的那段邏輯可以說是超級陌生、完全不熟；第二個是公司的 spring boot 專案版本非常新（v3.4），因此在網路上或 GPT 等 AI 服務幾乎沒有可以參考的範本或教學。前前後後花了近一個月才把功能搞定，特別寫一篇文章來記錄一下。

## 最終實現結果

先說明一下最後的實現結果：當我們在 cmd 打上 mvn clean package 指令之後，程式會自動生成一份「部分加密」的 jar 檔案。具體來說：在 jar 檔的 BOOT-INF/classes 資料夾裡，存在著部分已 .class 為副檔名的普通檔案，以及部分已自定義副檔名為結尾的加密檔案。當然了，電腦在執行這份 jar 的時候也不會自動忽略這接加密檔案：當我們使用 java -jar xxx.jar 嘗試運行這份 spring boot 的 jar 檔之後，那些加密檔案仍可以自動被系統解密並視為正常的 .class 檔，同時如果檔案原本就帶有 @Component 的話，也會被定義到正常的 Bean 裡面，作為被 spring boot 容器管理的物件之一。詳細的專案原始碼可以參考 [github 上的程式細節](https://github.com/pax131072/spring-boot-fat-jar-encryption-protection-tool)。至於背後的邏輯是怎麼實現的，可以看下面的說明：

## Fat Jar Editor

要完成這項功能，首先要從「製作出一個帶有部分加密檔案的 jar」做起。這裡研究了一下之後，發現有 2 種方式算是比較可行的道路：第一種是在 pom.xml 裡面移除 spring boot maven plugin 套件，並手動繼承一個 Spring boot 的 Repackager，從中改寫裡面的部分方法，然後讓 maven 的打包執行這個自定義的 repackage 插件。這個應該是官方推薦的改裝流程(? 因為多數環節還是繼承原始的 Repackager 邏輯，所以不會跟預設的 spring boot 打包相差甚遠。但因為這方法會介入比較多的方法跟環節，我個人比較不喜歡，所以最後用的是第二種方法；第二種方法同樣是自定義插件，只不過插件的執行時機從「替換原本的 repackage」調整為「預設的 repackage」之後：手動複製一個已經打包好的 fat jar，並在複製的過程中對部分檔案進行加密或搬移就是這種方式的運作原理。優點是邏輯相對單純，但缺點是可能會有比較多的魔術字串依賴，可維護性會再低一點。

## JarLauncher / ClassLoader

調整好 spring boot 打包完的 fat-jar 之後，再來要手動改寫系統的 ClassLoader，原因應該也很好理解：因為有部分的 .class 檔案已經被我們加密了，所以通常的 class loader 是沒辦法正常讀取這些檔案的 byte code 的。那在 spring boot 裡面這一點會再麻煩一點點：ClassLoader 主要是由一個叫做 JarLauncher 的組件所創建，JarLauncher 則是我們在打上 java -jar xxx.jar 之後程式真正會執行的第一個程式。同時根據 java 的執行規則，打包後的 jar 檔會去執行 MANIFEST.MF 裡面，Main-class 的 main() 方法。所以在 Fat Jar Editor 裡面同時也需要調整這裡的 Main-class 參數（改成自己的 JarLauncher），然後 JarLauncher 再去 new 一個自定義的 ClassLoader，再再從 ClassLoader 裡面覆寫 findClass() 方法：當原本的 find class 無法順利運行的時候，先自動默認該檔案就是我們手動操作過的加密檔案，嘗試對它進行解密與載入。

## BeanDefinitionRegistryPostProcessor

最後一個要完成的元件是自定義的 BeanDefinigion 註冊器。因為根據正常的 spring boot 運作邏輯：在組件掃描的環節，spring boot 只會掃描以 .class 為副檔名的 java 檔案，並根據一系列條件判斷這份檔案是否需要從 bean definition 轉變為 bean。如果我們的加密檔案本身帶有 @Component 等相關註解，因為副檔名不為 class 的原因，這裡預設是不會被掃描進去的。所以要自己手寫一個註冊程序，讓自定義的加密檔案後續也能整合到 spring boot 的相關流程裡面。主要改動的內容有 3 個：第一個是擴充的 Post Processor 物件本身，第二個是 MetadataReader（原始的加密檔案資訊讀取到 spring 容器裡面），最後一個是 MetadataReaderFactory，因為 Post Processor 與 Reader 本身無法直接交互，中間需要透過 factory 連動，所以這裡還要多實作一個 Factory 類別。

至於範例給的程式裡面，還多了一個 Class Path Scanning Candidate Component Provider 的子類別，它是用來判斷加密檔案是不是有包含 @Component 註解的（如果有，才有機會在最後變成一個真正的 Bean，但還是要看 ConditionOnXxx 等判斷條件有沒有讓這些類別失效）如果在最前面的加密流程裡面，可以保證被加密的檔案本身不會是個 Bean，或者是不需要經過一些判斷之後才能決定它是不是 Bean，那這個 Candidate Component Provider 是可以不用實作的，直接拿預設的 prodiver 來用就可以，不會影響功能。

## 後記

spring boot 從 2.x 轉到 3.x 之後，有對 JarLauncher 和 ClassLoader 的實作與架構進行了大量改動，因此在網路上，加解密 fat jar 幾乎可以說是找不到任何的資料，就算是問 AI 或者是找到其他的相關資源，大多也是只提供 2.x 系列的實現方式，因此本篇文章記錄了一個以 spring boot 3.4 為基底、自定義加解密 fat jar 的功能實現邏輯，提供給自己或其他有需要作到相關功能的朋友做為參考。也因為是範例的原因，文章內部省略了部分的資安細節與單一職責的類別宣告方式，如果後續有需要將這項功能進行正式導入，可以再稍微調整一下裡面的實現方式，使得系統除了能擁有加解密的功能以外，還能夠擁有更好的程式架構，以維持原始專案的可讀性與可維護性。