---
title: 自定義 @QueryParamAlias 註解：Controller 的參數別名綁定實踐
date: 2025-07-10
tags: [java, spring-boot]
---

第一次實作自定義註解，回頭看感覺蠻好玩的，想說紀錄一下。

會想要自定義註解的原因主要是以下幾點：專案現行的後端系統因為一些歷史原因，某些 api 會需要攜帶大量參數作為請求（大概 15-20 個的那種大量），同時請求中還有 `update_latestuserLOGINinfo` 這樣不易閱讀的參數。由於歷史相依或上下游系統的限制，這些參數命名無法任意調整，同時專案目前對這些 api 請求的處理方式是用 @RequestParam 一個一個手動映射（或者是用 Map 統一蒐集，再在 controller 裡面自己轉換）。整體來說，無論是從可讀性還是可維護性的角度來看，Controller 這邊都存在大量的改進空間。所以這次的目標就是要針對這些難以閱讀與維護的 Controller 方法進行整理與改善，讓它們變得相對簡單與親近。

改進的方法主要朝著 2 個方向進行：對於 POST + application/json 的 api 請求來說，改寫方法其實很簡單：先把輸入參數封裝成唯一名稱的 RequestDTO，然後在 DTO 內部加上 @JsonProperty 註解進行請求參數的名稱映射，基本上就把問題解決掉了。但這種 api 算是少數，大多數的請求來是處在比較麻煩的那一邊。因為即使用了 DTO 封裝請求參數，@ModelAttribute 還是沒有辦法對參數名稱做自動改寫，加上自動映射本來是 @RequestParam 可以處理的功能，但 @RequestParam 無法像 @JsonProperty 那樣標註在 DTO 的欄位上，進而實現參數名稱的映射。因此這邊才需要自定義一個 @QueryParamAlias 註解，讓它可以標註在成員變數上，同時進行參數的名稱轉換。

``` Java
class FooApiRequestDTO {
    @QueryParamAlias("update_latestuserLOGINinfo")
    private String updateLatestUserLoginInfo;
}
```

首先需要實作註解。這是所有過程裡面最簡單的一步：定義一個 @interface，名稱叫做 QueryParamAlias，然後 Target 限制在 ElementType.FIELD（也就是類別裡面的屬性）上面，Retention 需要設定成 runtime（因為需要在 java 運行期間透過反射獲得相關的 Field 內容）。String value() 就沒甚麼好說的，註解需要一個可以存放 ("xxx") 的地方，原本想要取名叫做 alias 或 requestParamKey 的，但想想還是把它寫得簡單一點就好，不要做太多花俏的東西，以防自己（或者是以後接手的人）看不懂這個變數到底想要幹嘛，造成額外的維護成本。

```Java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface QueryParamAlias {
    String value();
}
```

再來，從 servlet 運進來的請求資料會經過一個叫做 Handler Method Argument Resolver 的元件進行資料綁定（詳情可以參考另外寫的[這一篇文章](#/article/spring-controller-param-binding-mechanism)）自定義註解因為不是 Spring MVC 的內建解析器，所以需要自己實作。實作的方法很簡單，創建一個類別，讓它繼承 HandlerMethodArgumentResolver，然後完成裡面的 supportsParameter() 和 resolveArgument() 就可以了。supportsParameter 的判斷同樣不難：只要檢查物件的 field 有沒有出現 @QueryParamAlias 就好，如果有就回傳 true（代表可以由這個解析器負責處理），沒有就回傳 false，把解析的工作交給其他 resolver。

resolveArgument 就稍微麻煩一點了：首先透過反射的機制取得一個 DTO 的 instance，然後歷遍 dto 的 fields，如果 field 上帶有 @QueryParamAlias 註解，就取得註解上的 value（也就是 ("...") 裡面的內容），然後把它當成 key，回頭查找請求的資料裡面有沒有這組 key-value？接著做型態轉換（getMapping 如果用 queryParams 的方式傳遞參數，即便是全數字也會在請求裡便自動變成 String），把參數對應到正確的型別與數值。之後等 DTO 裡全部的 field 都做完這些事情，再跑一次 @Valid 註解的驗證方法（所以這個註解可以支援 @Valid 的驗證邏輯）。這樣就做完一輪轉換了。

```Java
@Component
public  class QueryParamAliasAnnotationResolver implements HandlerMethodArgumentResolver {
    @Autowired
    @Lazy
    private ConversionService conversionService;

    @Override
    public boolean supportsParameter(@SuppressWarnings("null") MethodParameter parameter) {
        Class<?> paramType = parameter.getParameterType();
        for (Field field : paramType.getDeclaredFields()) {
            if (field.isAnnotationPresent(QueryParamAlias.class)) {
                return true;
            }
        }
        return false;
    }

    @SuppressWarnings("null")
    @Override
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
        Class<?> paramType = parameter.getParameterType();
        Object dto = paramType.getDeclaredConstructor().newInstance();
        Map<String, String[]> paramMap = webRequest.getParameterMap();

        for (Field field : paramType.getDeclaredFields()) {
            QueryParamAlias alias = field.getAnnotation(QueryParamAlias.class);
            if (alias != null) {
                String paramName = alias.value();
                String[] values = paramMap.get(paramName);
                if (values != null && values.length > 0) {
                    field.setAccessible(true);
                    String rawValue = values[0];
                    Object convertedValue = conversionService.convert(rawValue, field.getType());
                    field.set(dto, convertedValue);
                }
            }
        }

        // if has @Valid annotation
        if (parameter.hasParameterAnnotation(Valid.class) || parameter.hasParameterAnnotation(Validated.class)) {
            SpringValidatorAdapter validator = new SpringValidatorAdapter(jakarta.validation.Validation.buildDefaultValidatorFactory().getValidator());
            BeanPropertyBindingResult result = new BeanPropertyBindingResult(dto, parameter.getParameterName());
            validator.validate(dto, result);

            if (result.hasErrors()) {
                throw new org.springframework.web.bind.MethodArgumentNotValidException(parameter, result);
            }
        }
        return dto;
    }
}
```

最後，我們要把做好的解析器也綁到 Handler Method Argument Resolver 的 list 上。如果我們是用 addArgumentResolvers() 方法把解析器綁到 0 的位置，它最後還是不會吃到我們的 resolver，因為 addArgumentResolvers 實際上調整的是自定義解析器的先後順序，即便你把它擺在最頭，它也只會是在「自定義解析器的初始位置」而不是「整個 Resolvers 的初始位置」。一個比較可行的調整方法是改寫整個 WebMvcRegistrations，在 getRequestMappingHandlerAdapter() 方法裡面手動調整自定義解析器的順序為「所有 resolver 的初始位置」，這樣就可以確保解析器會最先嘗試處理參數，而不是讓其他解析器搶先完成請求資料的綁定：

```Java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private QueryParamAliasAnnotationResolver queryParamAliasAnnotationResolver;

    @Bean
    public WebMvcRegistrations webMvcRegistrations() {
        return new WebMvcRegistrations() {
            @Override
            public RequestMappingHandlerAdapter getRequestMappingHandlerAdapter() {
                return new RequestMappingHandlerAdapter() {
                    @Override
                    public void afterPropertiesSet() {
                        super.afterPropertiesSet();
                        List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();
                        resolvers.add(queryParamAliasAnnotationResolver);
                        resolvers.addAll(getArgumentResolvers());
                        setArgumentResolvers(resolvers);
                    }
                };
            }
        };
    }
}
```

到這邊為止，請求就可以正確的被賦值了。註解支援的範圍有 GET（Query String）、POST + application/x-www-form-urlencoded（表單請求）、還有 POST + multipart/form-data（檔案資料上傳）3 種地方，POST + application/json 原本就可以被 @JsonProperty 進行映射，所以不用處理，專案裡其餘的 api 請求也都是以其他 3 種傳輸方式為主。後面就可以透過這個 @QueryParamAlias 註解實現請求參數的名稱映射與 DTO 綁定，後端不再需要因為前端參數的 dirty key 而汙染系統的變數名稱。最後用起來的樣子大概就像下面這樣（有對一些關鍵參數做了模糊處理）。

```Java
package com.example.demo.dto;

import com.example.demo.annotation.QueryParamAlias;
import lombok.Data;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Size;

@Data
public class UserLoginTrackRequestDTO {

    @QueryParamAlias("useracctid")
    @NotBlank(message = "使用者帳號不可為空")
    @Size(max = 32, message = "使用者帳號長度不得超過 32 字元")
    private String userAcctId;

    @QueryParamAlias("client_APPversion")
    @NotBlank(message = "App 版本不可為空")
    @Pattern(regexp = "^v\\d+(\\.\\d+){0,2}$", message = "App 版本格式錯誤，範例：v1.2.3")
    private String ClientAppVersion;

    @QueryParamAlias("logintimeUTC")
    @NotBlank(message = "登入時間不可為空")
    @Pattern(regexp = "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}(:\\d{2})?Z$", message = "時間需為 ISO UTC 格式")
    private String loginTimeUTC;

    @QueryParamAlias("update_latestuserLOGINinfo")
    @NotNull(message = "是否更新登入紀錄不可為空")
    private Boolean updateLatestUserLoginInfo;
}
```