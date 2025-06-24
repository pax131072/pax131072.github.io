---
title: HandlerExceptionResolver：Spring MVC 的全域例外處理機制
date: 2025-06-23
tags: [java, spring-boot]
---

上篇文章說明了 Spring MVC 的參數綁定機制，這邊文章來講例外處理機制。

在 Spring MVC 裡面，Http 請求的核心處理元件是 DispatsherServlet。網路上如果用圖片搜尋「DispatsherServlet」，會有很大的機率找到跟下面很相似的圖片，這張圖片在講的就是 DispatsherServlet 的核心調度方法：`doDispatch()`。以 2.7.18 版本的 spring boot 為例（其他版本的實作框架應該也幾乎相同或完全沒改動），doDispatch 在收到請求之後，依序會執行 multipart 檢查、透過 HandlerMapping 找到 handler、透過 HandlerAdapter 找到 adapter、請求前處理、執行請求（Controller 和 Controller 後的業務邏輯都在這裡）、後請求處理、與回傳請求結果。而這過程中的任一環節如果出現例外，且該例外一路被向上拋回 doDispatch 這邊，就會觸發到 Spring MVC 的核心例外處理機制。

![DispatcherServlet.doDispatch() 流程圖](images/spring-mvc-handler-exception-resolver/dispatcherServlet.png)

Spring MVC 的例外處理機制主要由一個名叫 **HandlerExceptionResolver** 的元件負責，HandlerExceptionResolver 本身是一個接口，該接口只包含一個 `resolveException` 方法，實作接口的例外處理類別需要自行判斷「當例外被丟到我這邊之後，我該不該處理它？如果要，又該怎麼處理？」底層的實作邏輯本質上採用的是責任鏈模式，也就是說：當 spring 系統啟動之後，系統就會掃瞄並建立一條 exception resolver chain，而當例外出現時，系統就會利用這條 chain 依序嘗試處理例外，如果鏈上靠前的處理元件可以 handle 這個例外，就會把它處理掉，不繼續往下傳遞。處理的邏輯同樣在 doDispatch 裡面，不過需要稍微點進幾個方法裡：doDispatch > processDispatchResult > processHandlerException > resolveException。

![resolveException 的責任鏈程式](images/spring-mvc-handler-exception-resolver/processHandlerException.png)

如果我們把斷點打在這個位置，並且嘗試運行一個會拋出例外的 Controller 方法，我們會發現 resolvers 裡面包含著兩個元件：分別是 DefaultErrorAttributes 和 HandlerExceptionResolverComposites。在實際的情況下，DefaultErrorAttributes 並不會去處理例外，它所做的事情只是記錄與例外有關的相關參數（像是 timestamp、status、error、path ...等），並把它記錄在一個叫做 ErrorAttributes 的物件裡面，如果其他的處理器有需要提取與這個 exception 相關的參數，它們就不用再從四處撈出這些散落的資料，只需要呼叫物件，然後把想要的屬性拿出來即可。

![handlerExceptionResolverList](images/spring-mvc-handler-exception-resolver/handler-exception-resolver-list.png)

![DefaultErrorAttributes 不參與例外處理](images/spring-mvc-handler-exception-resolver/defaultErrorAttributes-resolveException.png)

HandlerExceptionResolverComposites 則是真正有在處理例外的元件。同樣，在預設情況下，這個物件裡面會維護一個 resolvers 陣列，裡面會存放真正可以處理例外的 resolver 們。通常來說，這個陣列的元件依序會是 ExceptionHandlerExceptionResolvers、ResponseStatusExceptionResolvers、和 DefaultHandlerExceptionResolvers，這裡同樣也是責任鏈模式的實作方式，如果靠前的 resolver 能夠把例外處理掉，就不需要讓後面的 resolvers 動手。通常來說，如果系統內有做統一的例外處理（也就是有 @ControllerAdvice 註解的類別 + @ExceptionHander 註解的方法）第一個解析器就會處理完所有內容。

![HandlerExceptionResolverComposite.java](images/spring-mvc-handler-exception-resolver/handler-exception-resolver-composite.png)

ExceptionHandlerExceptionResolvers 顧名思義，就是「帶有 @ExceptionHandler 這個註解的錯誤處理方法。」在系統運行之初，Spring 會先掃描範圍內所有帶有 @ExceptionHandler 的註解，並把它們存在一個叫做 exceptionHandlerAdviceCache 的 Map 裡面，Map 的 key 存放方法可以處理的例外類別，也就是 `@ExceptionHandler(value = ...)` 填入的值，value 則是加上註解的方法本身。當例外發生時，ExceptionHandlerExceptionResolvers 就會從 exceptionHandlerAdviceCache 嘗試找到能處理例外的方法，如果 map 找不到完整的類別對應，就會再一路往上找例外的父類有沒有 @ExceptionHandler ...，而當查找成功，就交由那個方法進行例外處理。

ResponseStatusExceptionResolvers 放到現在的時代來看，應該已經幾乎沒有人再使用了。當方法（或類別）上面帶有 `@ResponseStatus(HttpStatus.xxx)` 註解，或者是業務邏輯拋出 ResponseStatusException，就有可能被這個 Resolver 處理到。這個 Resolver 的劣勢在於它只能處理 http status code 和 error message，不支援更加複雜的錯誤回傳格式。同時即便把 reasoon 調整成 JSON 字串，後續的處理上也沒有辦法把這個字串解析成 application/json。這對於現今比較常使用 REST API 的環境來說，算是限制非常多的類別，因此比起使用這個 Resolver，還是更推薦使用 @ControllerAdvice + @ExceptionHandler 的例外處理方式。

DefaultHandlerExceptionResolver 是 Spring MVC 中，HandlerExceptionResolverComposite 裡的最後一個解析器，主要的作用是作為「兜底機制（fallback）」，處理那一些前面的解析器不能解決的例外類別，通常這是指不能被 @ExceptionHandler 解決的那些例外。對這些例外，它會設定適當的 HTTP 狀態碼（如 405、415、400），再將狀態碼送回客戶端。通常 DefaultHandlerExceptionResolver 會配合 BasicErrorController，後者會將錯誤的訊息以簡單的 JSON 格式組裝起來，裡面會包含 timestamp、status、error、path 等資訊，對於沒有實作統一例外處理的 Spring 系統來說，應該蠻常看到的。

![Spring MVC 預設的例外回傳畫面](images/spring-mvc-handler-exception-resolver/default-springmvc-error-response.png)

---

#### 參考資料：
##### [web 九大組件之 HandlerExceptionResolver 異常處理器使用詳解](https://blog.csdn.net/f641385712/article/details/101840833)
##### [Spring boot Web MVC：錯誤屬性處理工具 DefaultErrorAttributes](https://blog.csdn.net/andy_zhang2007/article/details/89048854)