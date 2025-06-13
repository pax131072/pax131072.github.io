---
title: 寫了一個 Laravel 風格的 Java Map Validator
date: 2025-06-12
tags: [java, spring-boot]
---

先說結論，我還是覺得 DTO 比較好用。

這 2 個月在熟悉公司的程式和風格，整體而言，非常不適應。主要是目前主管的一些開發習慣跟我之前的經驗相差甚遠。主管不喜歡用 DTO，她覺得把參數額外包裝成一個類別（或物件）是很反直覺得事情，所以限制參數都只能用 `Map<String, Object>` 傳遞。這可能跟公司之前是以 PHP 開發產品有關？在我的認知裡，Java 以外的語言好像很少用 DTO 封裝參數，原因未知。不過以我的角度來看，我覺得 DTO 雖然需要額外維護一些類別、物件 ...等資料，但其可讀性和可維護性相對於直接使用 Map 或一大串 8 個 10 個放進 Method 裡面的參數來說，算是相對友善很多了吧？Anyway，既然主管不喜歡 DTO，我又需要一個**可以簡單快速知道 Map 裡面包裝了哪些參數**的特性，就只能自己手動寫一個 Validation 了。

在開發功能之前，首先要先定義好我要的功能是什麼？它長什麼樣子？又可以做到哪些事情？具體來說，我的目標是開發一個 validate 方法，該方法可以輸入一個 params（就是 method 接收到的入參）和一個 rules（就是 params 裡面的 k-v 要遵循哪些規則去做驗證或檢查？如果檢查不符又該如何處理）。可以的話，我希望這個 validate 方法用起來盡可能跟 php 的 validation 造型相似，就像是下面的這個樣子：

```php
Validator::make($request -> all(), [
    'title' => 'required|unique:posts|max:255',
    'body' => 'required',
]) -> validate();
```

所以思考之後，決定方法用起來要長這樣：

```Java
MapUtils.makeValidator(request, Map.ofEntries(
    Map.entry("title", "required|unique:posts|max:255"),
    Map.entry("body", "required")
)).validate();
```

思考的過程大概如下，首先不考慮把類別直接寫成明顯的 Validator（或者說外顯地用 Validator 做驗證）而是包裝在 MapUtils 工具類裡面，原因有兩個：1 是因為 Spring Framework 本身有 @Validated 跟 @Valid 關鍵字，如果這邊直接用 Validator，可能會被人誤會是 Spring 其他功能之類的；2 是因為該方法本來就是專門給 Map 使用的，所以權衡之下就改用 MapUtils。方法名稱這邊也沒有直接沿用 `make`，原因同樣是語意問題（MapUtils.make 容易聯想成創建 Map 之類）。makeValidator 方法會回傳一個自定義的 Validator 類別，而該類又可以提供一個 validate 方法供檢驗用，同樣會回傳經過驗證且清洗後的資料，這邊就完全維持 laravel 的模式。

結果如下：

```Java
public class MapUtils {
    public static Validator makeValidator(final Map<String, Object> data, final Map<String, String> rules) {
        return new Validator(data, rules);
    }
}
```

---

```Java
public class Validator {
    private Map<String, FieldDefinition> fields;

    @AllArgsConstructor
    @Getter
    private class FieldDefinition {
        private final Object value;
        private final List<Rule> rules;
    }

    public Validator(final Map<String, Object> data, final Map<String, String> rules) {
        this.fields = new HashMap<>();
        for (Map.Entry<String, String> entry: rules.entrySet()) {
            String key = entry.getKey();
            String ruleString = entry.getValue();
            Object value = data.get(key);
            this.fields.put(key, new FieldDefinition(value, parseRules(ruleString)));
        }
    }

    public Map<String, Object> validate() {
        Map<String, Object> validated = new HashMap<>();

        for (Map.Entry<String, FieldDefinition> entry : fields.entrySet()) {
            String key = entry.getKey();
            Object value = entry.getValue().getValue();
            List<Rule> rules = entry.getValue().getRules();

            boolean required = false;
            for (Rule rule: rules) {
                if (rule instanceof RequiredRule) required = true;
                value = rule.apply(key, value);
            }
            if (value != null || required) {
                validated.put(key, value);
            }
        }

        return validated;
    }

    private List<Rule> parseRules(String ruleStr) {
        List<Rule> rules = new ArrayList<>();
        for (String token : ruleStr.split("\\|")) {
            if (token.equals("required"))
                rules.add(new RequiredRule());
            else if (token.equals("nullable"))
                rules.add(new NullableRule());
            else if (token.equals("string"))
                rules.add(new StringRule());
            else if (token.equals("integer"))
                rules.add(new IntegerRule());
            else if (token.startsWith("between:"))
                rules.add(new BetweenRule(token.substring(8).split(",")));
            else if (token.startsWith("date_format:"))
                rules.add(new DateFormatRule(token.substring("date_format:".length())));
            else
                throw new IllegalArgumentException("Unsupported rule: " + token);
        }
        return rules;
    }
}
```

---

```Java
public abstract class Rule {
    public abstract Object apply(String key, Object value);
}
```

主要的程式細節集中在 `Validator` 類別上，建構子提供 `Validator(data, rules)`，前者用來存放要驗證的 Map，後者用來記錄要存放的規則，Validator 只有一個成員變數（fidles），用來存放 key, value 和 rules。rules 做了一層抽象（`Rule` 類別），實際要用哪一個規則需要由 rules 的字串判斷，判斷邏輯放在 parseRules，目前的做法是把它當成一個簡單工廠，有新規則就無限加進去，之後如果有逐漸擴大規模再考慮拆分。每個 Rule 裡面只需要提供 apply 方法，該方法用來檢查並回傳 value，這裡就跟 Laravel 不一樣了：如果遇到 Integer, String 這種驗證規則，預設是會直接幫你做轉型的（意思是 "123" value 在經過驗證後能會變成 123），主要是為了以後如果願意換成 DTO 的話，這邊的 Map 資料可以直接拿過來注入，但缺點就是會有隱性錯誤的問題。

```Java
public class IntegerRule extends Rule {
    @Override
    public Object apply(String key, Object value) {
        if (value == null) return null;
        try {
            // 自動轉型，有機會出現非預期的錯誤
            return Integer.parseInt(value.toString());
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("[" + key + "] must be an integer.");
        }
    }
}
```

最後實際用起來大概像下面這樣：

```Java
@RestController
@RequestMapping("test")
@Slf4j
public class TestController {

    @PostMapping("/foo")
    public ResponseEntity<?> foo(@RequestBody Map<String, Object> request) {
        // 檢查 map 元素
        request = MapUtils.makeValidator(request, Map.ofEntries(
            Map.entry("account", "required|string"),
            Map.entry("password", "required|string"),
            Map.entry("logintime", "required|date_format:Y-m-d H:i:s"),
            Map.entry("role", "required|string"),
            Map.entry("tokenExpire", "required|integer|between:0,9999")
        )).validate();

        // 作一些其他操作，像是透過參數創建 token 之類的
        Map<String, Object> token = tokenService.createToken(request)
        return ResponseEntity.status(HttpStatus.OK).body(token);
    }
}
```

再來是後續計畫：根據目前公司的程式規模來看，寫成這樣應該就夠用了，但之後有時間的話，我還想加入一些其他的新功能（或者說補充更多 Laravel Validator 有的功能），像是更多的規則，例如 regex, boolean, array, email...等；可以巢狀檢查嵌套的 key, value 參數；max, min ...等驗證規則可以根據 String, Number, Collection ...等不同的資料型態有不同的檢查邏輯；可以支援更加個性化的 error message 訊息；以及可以同時檢查所有規則欄位，並一給出所有出錯的參數以及未符合規則 ...等。但如果可以的話，還是希望公司可以引進 DTO 的機制，用 @Valid 和 @Validated 做驗證，會輕鬆很多。