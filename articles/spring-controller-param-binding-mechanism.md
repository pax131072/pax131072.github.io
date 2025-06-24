---
title: HandlerMethodArgumentResolver：Controller 的參數綁定原理
date: 2025-06-19
tags: [java, spring-boot]
---

有些能夠簡單使用的東西，背後都封裝了大量的細節與努力。

這幾天在公司處理 API Controller 的時候，遇到了一個小問題：當請求是以 post 方法打進來，同時 request body 是以 application/json 的格式傳遞資料的時候，Controller 這邊如果沒有對需要自動綁定的參數加上 @RequestBody 註解，那綁定機制就會失效。而解決方法除了加上註解以外，把接收的參數從原始資料型態轉為 DTO 物件、或者是將請求方法改成 GET 也能做到。這就讓我有點好奇？在 Spring 底層裡面到底是怎麼對這些參數資料做自動綁定的？稍微研究了一下之後，覺得這個東西蠻值得寫下來的，所以就寫了這篇文章記錄一下。

```bash
curl -X POST http://127.0.0.1:8080/test -H "Content-Type: application/json" -d '{"name": "Alice", "age": 20}'
```
```Java
// name 和 age 寫成這樣無法自動綁定 Alice 和 20
@PostMapping("/post")
public ResponseEntity<?> post(String name, int age) {
    // do something ...
}
```

![post 請求失敗](images/spring-controller-param-binding-mechanism/post-request-failed.png)

先直接破題：Spring 裡面對 Controller 參數進行請求資料綁定的核心元件叫做 **HandlerMethodArgumentResolver**。綁定的行為會發生在「HandlerMapping 已經找到對應的 HandlerAdapter，並且 HandlerAdapter 在調用 controller 方法之前。」Handler Method Argument Resolver 實質上是一個 Interface，裡面包含了兩個需要 implements 的方法：分別是 supportsParameter 和 resolveArgument。前者決定這個 ArgumentResolver 物件（後稱解析器）支不支援解析特定型態的方法參數，後者則決定解析器裡面的解析與綁定細節要怎麼進行。因此，參數能不能被自動綁定某個數值，取決於參數使用了哪種解析器，以及該解析器支援什麼方式的資料綁定。

```Java
package org.springframework.web.method.support;

public interface HandlerMethodArgumentResolver {

	boolean supportsParameter(MethodParameter parameter);

	@Nullable
	Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}
```

在真實的場景下，系統啟動之初，Spring 就會把所有的解析器創建出來，並以 List 的方式儲存。系統首先會掃描 Contrtoller 方法中的所有參數，讓參數們依次歷遍這個 resolver List，並記錄第一個能匹配自己狀態的的解析器（因此 resolver list 的元素順序會影響參數對解析器的挑選結果）。而在之後，當請求被送進 Spring 系統時，Spring 首先會定位到哪個 Controller 方法要處理這項請求，之後就會歷遍該方法的所有參數，並檢查請求裡的資料可不可以透過對應的解析器將資料自動注入進來。實作（或繼承）Handler Method Argument Resolver 的類別非常非常多，我們可以透過下面的程式列出系統的 ResolverList 長什麼樣子。以 2.7.18 版本的 spring-boot 為例：

```Java
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;

@GetMapping("/resolvers")
public ResponseEntity<?> printResolvers() throws JsonProcessingException {
    List<HandlerMethodArgumentResolver> resolvers = handlerAdapter.getArgumentResolvers();
    
    List<String> result = new ArrayList<>();
    for (HandlerMethodArgumentResolver resolver : resolvers) {
        result.add(resolver.getClass().getName());
    }

    return ResponseEntity.status(HttpStatus.OK).body(objectMapper.writeValueAsString(result));
}
```
![ArgumentResolvers](images/spring-controller-param-binding-mechanism/argumentResolvers.png)

大致上來說，Resolvers 可以依照是否依賴於參數的註解與資料型態分成 3 種類型：第一種是「基於註解的 Argument Resolver」，這類 Resolver 會根據特定註解來判斷是否適用，通常會搭配明確的注解條件，常見範例有：@RequestParam, @RequestHeader, @RequestBody, @PathVariable, @CookieValue 或 @ModelAttribute，這些註解各自會對應一種（或多種）不同的解析器，像是 @RequestParam 對應到 RequestParamMethodArgumentResolver、@RequestHeader 對應到 RequestHeaderMethodArgumentResolver、或者是 @RequestBody 對應到 RequestResponseBodyMethodProcessor...等。

第二種是基於資料型態的 Argument resolvers。這類的解析器不會依賴任何註解，而是單純依據參數的資料型態來判斷是否適用。當 Spring 檢查參數時，只要參數型別與解析器支援的型別相符，就會選中該解析器。值得注意的是，除了與 Map 類型相關的解析器（例如 RequestParam**Map**MethodArgumentResolver, RequestHeader**Map**MethodArgumentResolver, ...等）之外，其餘與資料型態相關的解析器（像是支援 HttpServletRequest 類型的 ServletRequestMethodArgumentResolver、或者是支援 HttpEntityMethodProcessor 類型的 HttpEntityMethodProcessor ...等）在現今環境的 spring 開發上已經很少遇到了。

最後一種是不依賴於任何東西，只要不加上註解，讓幾乎所有參數都可以使用的預設解析器。預設的解析器有分為兩種，如果參數的資料型態是簡單型別（像是 int/Integer, float/Float, char 或 String ...等）預設的解析器會是 RequestParamMethodArgumentResolver。但如果參數的資料型態是非簡單型別（像是各種不同種類跟樣貌的 DTO），就會改用 ServletModelAttributeMethodProcessor 作為預設的解析器。基本上，在 spring 的邏輯裡面，每一個參數最終都會對應到一個負責的解析器，如果有參數沒有辦法找到對應的 resolver 進行參數解析，請求在打進來的時候就會直接 throw new IllegalArgumentExceptoin.

![handlerMethodArgumentResolverComposite](images/spring-controller-param-binding-mechanism/handlerMethodArgumentResolverComposite.png)

說完了，我們回到最一開始的問題：加上 PostMapping 註解的方法為什麼不能單純用 (String name, int age) 這樣的方式接收 application/json 的參數？又為什麼把 PostMapping 改成 GetMapping，或者是把 name 跟 age 加上 @RequestBody 的註解就可以接收到了？原因很簡單：在預設的情況下，簡單參數類型（name, age）是 RequestParamMethodArgumentResolver 負責解析的，同時 **RequestParamMethodArgumentResolver 並不支援「將資料從 application/json 中的資料自動取出與賦值」這項功能**。RequestParamMethodArgumentResolver 只支援 2 種取值方式：分別是 url 後面的 query-string（主要用於 get 方法），以及用於 Post 的 application/x-www-form-urlencoded。

所以解法很簡單：要嘛是把資料從 json 移出來，放到 form 表單裡面，要嘛是乾脆連 form 表單也不要用了，直接把參數串接到 api 請求後面的 query-string 上面也行（但對 Post 方法來說幾乎不這樣用就是）。不然也可以在後端上面做修改：首先是把 PostMapping 改為 GetMapping（因為改成 GetMapping 之後，參數習慣上就會放在 query-string 上面，算是誤打誤撞的一種解法）；另一種方法是對 name 和 age 都加上 @RequestBody 註解，因為加上註解之後，解析的工作會改由 RequestResponseBodyMethodProcessor 這個參數解析器負責，該解析器的取值方式就是「從 application/json 取值，並自動注入到對應參數當中。」

最後，分享一個自己對 RequestParamMethodArgumentResolver 做追蹤的斷點截圖，給想要實際跑一次的朋友參考。按照下面斷點的打法，應該通 Get 請求（或 Post + form 表單）都會成功走過一次。細節的 code trace 這邊就不拉出來講了，比起看我貼的一堆 code trace 圖片，我相信手動追蹤一次帶來的印象應該會更加深刻。而且還有機會發現一些更加有趣的議題！像是 RequestBody 一開始的內容不會被預先載入，只有「有解析器需要撈取 body」資料時，才會做 lazy loading；或者是 DTO 雖然有沒有加 @ModelAttribute 用的是不同解析器，但底層都是用同樣的解析邏輯 ...等。

![請求之前的相關追蹤斷點](images/spring-controller-param-binding-mechanism/resolver-code-trace.png)

---

#### 參考資料：
##### [Controller方法入参自动封装器（将参数parameter解析为值）](https://fangshixiang.blog.csdn.net/article/details/98989698)