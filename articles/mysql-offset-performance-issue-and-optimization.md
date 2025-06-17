---
title: MySQL OFFSET 的效能問題與改進方法
date: 2025-06-16
tags: [mysql]
---

這幾天在回頭補 MySQL 的相關基礎，遇到一個值得記錄的小細節。

先說結論：MySQL 的 OFFSET 指令在遇到需要回表的情況下，並不會直接跳過前 n 筆資料，而是會先全數回表撈取，再把前 n 筆資料捨棄，這在分頁操作上來說，如果沒有處理好，很容易遇到「越後面的分頁 loading 越慢」的問題。解法的話 ...可以使用 Keyset Pagination 把 OFFSET n 調整成 WHERE id > n 來快速定位到需要回傳的初始資料；另一種方式是用覆蓋索引，讓查詢的資料本身就存在在次級索引裡，不須要回表；但如果需要回傳的資料量大，使用覆蓋索引的解法不實際，這時候也可以改用 join 子查詢替代回表，先找出資料 id，再透過 id 回表撈資料，也能達到不錯的性能優化。

```
+----------+------------------+-----------------------------------+
|  column  | data type        | description                       |
+----------+------------------+-----------------------------------+
| id       | INT              | PK (clustered index)              |
| name     | VARCHAR(100)     | secondary index (`name, phone`)   |
| address  | VARCHAR(150)     | secondary index (`address`)       |
| phone    | VARCHAR(20)      |                                   |
| salary   | DECIMAL(10,2)    | secondary index (`salary`)        |
| ext      | VARCHAR(200)     | extended data                     |
+----------+------------------+-----------------------------------+
```

考慮到以下情境：系統內存在一個 `employee` 資料表，該表包含幾個與員工有關的不同欄位，像是 `id`, `name`, `address` ...等，其中 id 是主鍵（自增且唯一），而索引有 (name, phone), (address) 和 (salary) 三種。查詢的內容如下：「列出所有薪水大於 4 萬塊的員工資料，支援分頁查詢，一頁 20 筆，當前需要第 10 頁的內容。」很明顯地，在這個情境下，使用 salary 進行 WHERE 篩選是一個相對好的做法，經過簡單的計算，我們可以知道它需要的是第 180 ~ 200 筆資料，也就是說，我們使用下列的 SQL 語法達到情境所需要的內容：

```SQL
SELECT *
FROM   employee
WHERE  salary > 40000
ORDER  BY salary,
          id
LIMIT  180, 20;
```

考慮到不同人可能有相同的薪水面額，因此在排序的時候除了按照 salary 排列以外，也用 id 進行二次排序。接著我們得到預期結果，沒有任何問題，然後要求改成了需要第 100 頁的內容、第 1,000 頁的內容、第 10,000 頁的內容 ...，問題就發生了。如果我們仔細記錄每一次的 SQL 執行時間，我們就會發現：當頁數變得愈發龐大的時候，SQL 的執行效率就會越來越慢。這個問題在 OFFSET 比較小的時候可能還看不太出來，但當數字跨越到一定的量級之後，延遲的狀況就會變得非常明顯，就像下面這樣：

![offset 180 查詢結果](images/mysql-offset-performance-issue-and-optimization/offset-180.png)
![offset 1800 查詢結果](images/mysql-offset-performance-issue-and-optimization/offset-1800.png)
![offset 18000 查詢結果](images/mysql-offset-performance-issue-and-optimization/offset-18000.png)
![offset 180000 查詢結果](images/mysql-offset-performance-issue-and-optimization/offset-180000.png)

這是為什麼呢？要解釋這個問題，我們可以從另一個東西說起：

我們都知道：在 MySQL 中，為了讓資料表在**多數查詢情境下具備良好的查詢效率**，引入了「索引（Index）」這種資料結構。索引的主要目的是加速特定欄位的搜尋與排序，其運作方式類似書籍的目錄，讓資料庫能夠避免全表掃描（Full Table Scan），快速定位到符合條件的資料列。同時，因為索引僅是目錄，它本身並不儲存完整資料，通常僅包含索引欄位與對應的主鍵值。因此，當我們根據索引定位到結果後，仍需透過**回表（table lookup）**，依據主鍵去資料頁中撈取完整的欄位內容。舉例來說，若建立一個 name 和 phone 組成的複合索引，儲存在索引的資料就只有 name, phone 跟主鍵（id）。

![透過索引查找真實資料](images/mysql-offset-performance-issue-and-optimization/search-with-index.png)

接著，在 MySQL 裡，query 語句運行的過程中會伴隨許多[系統層級的狀態變數](https://dev.mysql.com/doc/refman/8.4/en/server-status-variables.html)參與運作，這些變數可用來協助觀察查詢行為與效能。今天我們的這個問題可以從以下 3 個狀態變數著手：首先是 **handler_read_key**，它代表「利用唯一索引來讀取資料的次數。」通常在使用主鍵（或唯一索引）直接查找資料時，就會用到它，是效率最佳的查詢方式之一；**handler_read_next** 代表「在索引樹逐筆掃描下一筆資料的次數。」如果我們在次級索引進行範圍掃描時，就會用到它。而 **handler_read_rnd_next** 則代表「在全表掃描中，一筆一筆讀取資料的次數。」是查找資料中，最最沒有效率的搜尋方式之一。因此，從時間的角度來看，我們需要做的事情就是**盡可能讓 handler_read_key, handler_read_next 提高，同時盡可能壓低 handler_read_rnd_next 的數值**。

搞懂之後，我們回頭看下 SQL 語句的狀態變數長什麼樣子：

![透過索引查找真實資料](images/mysql-offset-performance-issue-and-optimization/status-variable-with-origin-limit.png)

從這次查詢的執行情況中可以觀察到：Handler_read_rnd_next 的值約為 20 萬，Handler_read_key 跟 Handler_read_next 則分別是 1 跟 0。這代表 MySQL 實際讀取了約 20 萬筆資料列，也就是進行了等量次數的全表掃描。而這代表了兩件事：第 1 件事是 MySQL 並沒有利用 WHERE salary > 40000 的條件套用索引進行範圍過濾，而是直接進行了全表讀取；此外，第 2 件更重要的是，這個行為也隱含了使用 OFFSET 進行分頁的隱藏風險：**當使用 OFFSET N 時，MySQL 並不會主動跳過前 N 筆資料，而是會逐筆讀取前 N 筆資料，全數回表，並將前方的其捨棄後，再回傳第 N+1 到 N+LIMIT 筆資料。**

![OFFSET 的實際運作情況](images/mysql-offset-performance-issue-and-optimization/offset-work-flow.png)

## Keyset Pagination（游標分頁）

那要怎麼改善這個狀況呢？前面有說過，從實務經驗來看，解決這類 OFFSET 效能瓶頸的方法主要有三種：首先是改用 Keyset Pagination。既然 OFFSET 本身會導致大量資料被讀取後捨棄的問題，那就「不要用 OFFSET」，Keyset Pagination 的核心概念是：每次查詢都基於上一次查詢的結果作為游標，直接跳轉到下個起點，而非以行數偏移。因此，在 SQL 語句上我們就不會用 OFFSET 這個關鍵字，而是像下面這樣透過 WHERE 的方式去定位我們需要的資料：

```SQL
-- 每次分頁查詢時紀錄最後的 id，並於下次查詢將 ? 換成 id 數值
SELECT *
FROM   employee
WHERE  ( salary > 40000 )
        OR ( salary = 40000 AND id > ? )
ORDER  BY salary,
          id
LIMIT  20;
```

![OFFSET 優化方法 1](images/mysql-offset-performance-issue-and-optimization/optimize-offset-1.png)

這種方法的好處在於能夠有效避免全表掃描（Full Table Scan）。因為 WHERE 條件中的所有欄位都能命中索引的最左前綴（left-most prefix）或完整組合，所以 MySQL 的查詢優化器（Query Optimizer）在評估執行計劃時，會選擇走索引範圍掃描（Index Range Scan）或唯一查找（Index Lookup），進而跳過整張資料表的掃描，大幅提升查詢效能。但壞處是需要在程式端手動記憶每一次分頁查詢的 id 數值，此外，這種方法對於連續頁數的查找比較友善（第 x 頁找完之後，下一筆要找的是第 x+1 頁），如果要做跳頁查找的話（直接查詢第 rnd 頁的資料）Keyset Pagination 的實現會相對麻煩。

---

## 覆蓋索引（Covering Index）

另一種方法則是使用覆蓋索引，從附圖可以看到：經過調整之後的 Handler_read_rnd_next 數量已經變為 0 了。但相對地，Handler_read_next 的數量則從原本的 0 一路飆高到約 18 萬，這樣的改動是合理的嗎？其實可以，因為這樣的現象正好反映出我們成功將查詢從「全表掃描」優化成了「索引範圍掃描」。Handler_read_next 的數值飆升，代表 MySQL 在執行這段查詢時，是從索引樹的葉節點中依序往後掃描符合條件的資料進行線性掃描，在記憶體命中率與磁碟存取的效率上通常更優，因此是查詢效率的提升而非退步。換句話說，Handler_read_next 增加只是表示我們透過索引定位了大量資料列的位置，並非意味效能下降。

```SQL
-- 把 select 的欄位從 * 調整成單從索引就能找到的資料
SELECT id, salary
FROM   employee
WHERE  salary > 40000
ORDER  BY salary,
          id
LIMIT  180000, 20;
```

![OFFSET 優化方法 2](images/mysql-offset-performance-issue-and-optimization/optimize-offset-2.png)

不過覆蓋索引的這種做法也有一些限制：首先是「索引的資料量會隨著查詢需要的欄位變大。」將大量的欄位納入索引會造成索引膨脹，進而佔用更多的記憶體與磁碟空間；此外，因為以索引的方式查詢需要滿足最左前綴原則，這也代表查詢的資料必須基於一個固定的順序排列（且中間不能跳過），如果索引原本是以 (name, phone, address, mail) 的方式建立，查詢就只能查 (name) (name, phone) (name, phone, address) 或 (name, phone, address, mail)，其餘情況就不會走索引查詢；最後，它也不適合查詢需要大量欄位或回傳整筆資料的情境，因為在這種情況下，相對於走索引，SQL 仍然會進行回表，造成索引失效。

---

## JOIN + 子查詢

最後一種做法是子查詢 + join：這種方式的核心思路是，先用索引支援的條件查出符合分頁條件的主鍵（id）清單，接著再用這組 id 進行一次回表查詢，以此避免大範圍的回表與排序操作。這種寫法的好處在於：內層的查詢只會先取 id，這種情況就可以完全依賴的索引樹完成排序與分頁，效能是最好的。同時內層查詢結束之後，外層語句只會進行一次性的回表，也僅對少量的主鍵做實體資料撈取，避免 OFFSET 所造成的大量無效回表，這種作法也適合在資料欄位多、資料列大、或者是需要的資料相對浮動的情況（因為無論如何，內層首先都只會抓取 id 作為第一層查詢的結果）。

```SQL
-- 即使 LIMIT 的數字很大，也可以保證回表次數只依賴於 limit 的數量
SELECT e.*
FROM   employee e
       JOIN (SELECT id
             FROM   employee
             WHERE  salary > 40000
             ORDER  BY salary,
                       id
             LIMIT  180000, 10) AS sub
         ON e.id = sub.id; 
```

![OFFSET 優化方法 3](images/mysql-offset-performance-issue-and-optimization/optimize-offset-3.png)

但正如前兩種改善方法，join + 子查詢也不是沒有缺點的。這種解決方案的其中一個問題是他的語句結構相對複雜，應該不難發現：相較於前兩種調整方式，因為有子查詢的介入，使得 SQL 語句出現了一段不算小的嵌套，這會在無形中增加系統後續的可讀性與可維護性；此外，雖然回表次數被壓縮至一次，但如果內層查詢取得的 id 數量龐大，這些主鍵在被帶入外層進行查詢時，也會造成一定程度的傳遞與解析負擔；最後，這種查詢方法的排序同樣也要依賴於索引（因為內層還是用了一組覆蓋索引），如果需要能夠讓使用者自訂排序條件，就還是需要預先準備多種不同的索引排序方式。

總的來說，不同的優化方式各有其適用場景與限制：Keyset Pagination（基於主鍵偏移） 適用於固定排序、連續查詢的場景，效能最佳，但缺乏彈性，無法隨意跳頁。覆蓋索引 適合查詢欄位少、排序明確的場景，可避免回表，但索引結構需精心設計，且不利於查詢欄位動態變化。子查詢 + JOIN 回表 雖然語法較複雜，但能兼顧彈性與效能，是處理大資料量查詢與複雜排序的有效解法。實務上沒有「萬用解法」，每一種分頁與索引策略都必須根據資料表規模、查詢模式與系統可維護性綜合評估。在效能與可維護性之間取得平衡，才是良好資料庫設計的核心。