# 17 Web (Gorilla Web Toolkit)

Go 有內建撰寫 Web Server 的套件，可以自己實作一套 AP server。因為 web 還會用到版型與靜態資料，因此在專案目錄的配置建議如下：

```text
web
├── main.go
├── public
│   └── db.png
└── templates
    ├── layout.html
    ├── nav.html
    └── test.html
```

目錄說明

1. public: 放置靜態資料，實際運作上，AP server 可以不用處理靜態資料。
1. templates: 放置版型

## 主程式

Go 與 JAVA, PHP 等不同，不需要額外再使用 AP or Web Server，Go 內建 HTTP Server。

### main.go

```go { .line-numbers }
package main

import (
    "errors"
    "fmt"
    "html/template"
    "log"
    "net/http"
)

// MyData ...
type MyData struct {
    Title string
    Nav   string
    Data  interface{}
}

func generateHTML(w http.ResponseWriter, data interface{}, files ...string) {
    var tmp []string
    for _, f := range files {
        tmp = append(tmp, fmt.Sprintf("templates/%s.html", f))
    }

    tmpl := template.Must(template.ParseFiles(tmp...))
    tmpl.ExecuteTemplate(w, "layout", data)
}

func test(w http.ResponseWriter, r *http.Request) {
    data := &MyData{
        Title: "測試",
        Nav:   "test",
    }

    data.Data = struct {
        TestString   string
        SimpleString string
        TestStruct   struct{ A, B string }
        TestArray    []string
        TestMap      map[string]string
        Num1, Num2   int
        EmptyArray   []string
        ZeroInt      int
    }{
        `O'Reilly: How are <i>you</i>?`,
        "中文測試",
        struct{ A, B string }{"foo", "boo"},
        []string{"Hello", "World", "Test"},
        map[string]string{"A": "B", "abc": "DEF"},
        10,
        101,
        []string{},
        0,
    }

    tmpl, err := template.ParseFiles("templates/layout.html", "templates/nav.html", "templates/test.html")
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    tmpl.ExecuteTemplate(w, "layout", data)
}

func setCookie(w http.ResponseWriter, r *http.Request) {
    c1 := http.Cookie{
        Name:     "first_cookie",
        Value:    "Go Web Programming",
        HttpOnly: true,
    }
    c2 := http.Cookie{
        Name:     "second_cookie",
        Value:    "Manning Publications Co",
        HttpOnly: true,
    }
    http.SetCookie(w, &c1)
    http.SetCookie(w, &c2)
    w.WriteHeader(http.StatusOK)
}

func getCookie(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusAccepted)
    allCookie := r.Cookies()
    firstCookie, err := r.Cookie("first_cookie")
    if err != nil {
        if errors.Is(err, http.ErrNoCookie) {
            fmt.Fprintln(w, "first cookie not found")
        } else {
            fmt.Fprintln(w, "get first cookie failure")
        }
    } else {
        fmt.Fprintln(w, "first cookie:", firstCookie)
    }

    fmt.Fprintln(w, allCookie)
}

func toCyberon(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Location", "https://www.cyberon.com.tw/")
    w.WriteHeader(http.StatusFound)
}

func main() {
    mux := http.NewServeMux()
    files := http.FileServer(http.Dir("./public"))

    mux.Handle("/static/", http.StripPrefix("/static/", files))
    mux.HandleFunc("/", test)
    mux.HandleFunc("/set_cookie", setCookie)
    mux.HandleFunc("/get_cookie", getCookie)
    mux.HandleFunc("/cyberon", toCyberon)

    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err)
    }
}
```

### main.go 說明

1. import 必要的 package

    ```go { .line-numbers }
    import (
        ...
        "html/template"
        "net/http"
    )
    ```

    1. `"html/template"`: Go 的 template engine。可以直接修改版型，不用重啟系統。
    1. `"net/http"`: Http 模組

1. 實作 routing 機制：

    ```go { .line-numbers }
    mux := http.NewServeMux()
    files := http.FileServer(http.Dir("./public"))

    mux.Handle("/static/", http.StripPrefix("/static/", files))
    mux.HandleFunc("/", test)
    mux.HandleFunc("/set_cookie", setCookie)
    mux.HandleFunc("/get_cookie", getCookie)
    mux.HandleFunc("/cyberon", toCyberon)
    ```

    - http.NewServMux(): 產生 `ServMux` 物件，用來處理 url routing 的工作。

1. 處理靜態資料:

    ```go { .line-numbers }
    files := http.FileServer(http.Dir("./public"))
    mux.Handle("/static/", http.StripPrefix("/static/", files))
    ```

    1. 測試網址：[http://127.0.0.1:8080/static/db.png](http://127.0.0.1:8080/static/db.png)
    1. 指定檔案放的目錄：`http.FileServer(http.Dir("./public"))`
    1. 設定 `/static/` 開始的網址，指定到剛剛設定的 `FileServer`。`/static/` 之後的目錄與檔案，都會對應該 `public/`。因此 `public/` 的檔案目錄結構要與 `/static/` 相同。

1. 其他 URL 的 routing: 利用 `HandleFunc` 來設定 URL 與處理 function 的關係。以下的 sample，`/` 會執行 `test`, `/set_cookie` 會執行 `setCookie`

    ```go { .line-numbers }
    mux.HandleFunc("/", test)
    mux.HandleFunc("/set_cookie", setCookie)
    mux.HandleFunc("/get_cookie", getCookie)
    mux.HandleFunc("/cyberon", toCyberon)
    ```

1. 綁定 port 並啟動 web server

    ```go { .line-numbers }
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatal(err)
    }
    ```

1. 目前 web service 都會要求使用 HTTPS，Go 實作時，可用 `http.ListenAndServeTLS` 搭配 [CertBot](https://certbot.eff.org/) 取得憑証。

    - 在 `ListenAndServeTLS` 的 `certFile` 請用 CertBot 的 **fullchain** cert 檔案。
    - 因為在 android 上，會驗証是否是 fullchain cert。

## Handler 與 Request Parameter

Handler function 的定義：

```go { .line-numbers }
func name(w http.ResponseWriter, r *http.Request) {
    body
}
```

eg:

```go { .line-numbers }
func test(w http.ResponseWriter, r *http.Request) {
    ...
}
```

1. 可透過 `r.Method` 來判斷是 GET or POST 等
1. 透過 `r.PostFormValue` 來取得 POST 值。Go 有內建多種取 request 值的方式，整理如下：

| Field | Should call method first | parameters in URL | Form | URL encoded | Multipart (upload file)
| - | - | - | - | - | -
| Form | ParseForm | ✓ | ✓ | ✓ | -
| PostForm | Form | - | ✓ | ✓ | -
| MultipartForm | ParseMultipartForm | - | ✓ | - | ✓ |
| FormValue | NA | ✓ | ✓ | ✓ | -
| PostFormValue | NA | - | ✓ | ✓ | -

from: [Go Web Programming](https://www.manning.com/books/go-web-programming)

## Response Header

預設 response 的 status code 是 **200(OK)**，如果要修改 Status Code，可用 `w.WriteHeader`，eg: `w.WriteHeader(http.StatusAccepted)`。

eg:

```go {.line-numbers}
func toCyberon(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Location", "https://www.cyberon.com.tw/")
    w.WriteHeader(http.StatusFound)
}
```

1. 如果有需要異動 HEADER 時，`w.WriteHeader` 必須要在最後使用，否則修改的 HEADER 會無效。
1. 可以試著將上述的 `toCyberon` 內的程式順序交換，會發現回傳的 HEADER 沒有 `Location` 值。
1. 設定 Cookie 也是。因為 Cookie 是放在 HTTP 的 HEADER 內。

## Cookie

### Cookie Struct

```go {.line-numbers}
type Cookie struct {
    Name       string
    Value      string
    Path       string
    Domain     string
    Expires    time.Time
    RawExpires string
    MaxAge     int
    Secure     bool
    HttpOnly   bool
    Raw        string
    Unparsed   []string
}
```

### 設定/讀取 Cookie

```go
func setCookie(w http.ResponseWriter, r *http.Request) {
    c1 := http.Cookie{
        Name:     "first_cookie",
        Value:    "Go Web Programming",
        HttpOnly: true,
    }
    c2 := http.Cookie{
        Name:     "second_cookie",
        Value:    "Manning Publications Co",
        HttpOnly: true,
    }
    http.SetCookie(w, &c1)
    http.SetCookie(w, &c2)
    w.WriteHeader(http.StatusOK)
}

func getCookie(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusAccepted)
    allCookie := r.Cookies()
    firstCookie, err := r.Cookie("first_cookie")
    if err != nil {
        if errors.Is(err, http.ErrNoCookie) {
            fmt.Fprintln(w, "first cookie not found")
        } else {
            fmt.Fprintln(w, "get first cookie failure")
        }
    } else {
        fmt.Fprintln(w, "first cookie:", firstCookie)
    }

    fmt.Fprintln(w, allCookie)
}
```

1. 如果要修改回覆的 HTTP Status Code 時，`w.WriteHeader` 必須在要 `http.SetCookie` 之後。
1. 因為 Cookie 是寫在 HEADER 內。

## Templates

Go template engine 很好用，會自動依版型的內容，來自動做 escape 動作。使用 template engine 需要再學習它的語法。

Go html template 是利用 text template，因此相關的語法，要看 ["text/template"](https://golang.org/pkg/text/template/#pkg-index)

範例中，整理了我覺得常用的案例。

### Template 使用

```go { .line-numbers }
func generateHTML(w http.ResponseWriter, data interface{}, files ...string) {
    var tmp []string
    for _, f := range files {
        tmp = append(tmp, fmt.Sprintf("templates/%s.html", f))
    }

    tmpl := template.Must(template.ParseFiles(tmp...))
    tmpl.ExecuteTemplate(w, "layout", data)
}
```

### Template 說明

1. `template.ParseFiles(tmp...)`: 選擇會用到的版型檔案，要確認版型的路徑與檔案是否正確，並取得版型物件
1. `tmpl.ExecuteTemplate(w, "layout", data)`: 執行版型，並將版型會用的資料(`data`)帶入。其中 **layout** 是定義在版型內，指定要從那個區塊開始執行。

    - 在同一個版型檔案內，可以定義多個模版區塊 (也就是可以使用多組 `{{ define "NAME" }}`)，在 `ExecuteTemplate` 時，可以指定要使用那個區塊。

### 版型結構

範例的版型結構：

1. `layout.html`: 版型的主框。內含 `nav` (`{{ template "navbar" . }}`) 與  `content` (`{{ template "content" . }}`) 這個子版型。

    ```html { .line-numbers}
    {{ define "layout" }}
    <!DOCTYPE html>
    <html lang="en">
        <head>
            <meta charset="utf-8">
            <meta http-equiv="X-UA-Compatible" content="IE=9">
            <meta name="viewport" content="width=device-width, initial-scale=1">
            <title>Go Course Web - {{ .Title }}</title>
            <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css" integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKtu6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh" crossorigin="anonymous">
        </head>
        <body>
            {{ template "navbar" . }}
            <div class="container">
                {{ template "content" .Data }}
            </div>
            <script src="https://code.jquery.com/jquery-3.4.1.slim.min.js" integrity="sha384-J6qa4849blE2+poT4WnyKhv5vZF5SrPo0iEjwBvKU7imGFAV0wwj1yYfoRSJoZ+n" crossorigin="anonymous"></script>
            <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3UksdQRVvoxMfooAo" crossorigin="anonymous"></script>
            <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/js/bootstrap.min.js" integrity="sha384-wfSDF2E50Y2D1uUdj0O3uMBJnjuUD4Ih7YwaYd1iqfktj0Uod8GCExl3Og8ifwB6" crossorigin="anonymous"></script>
        </body>
    </html>
    {{ end }}
    ```

    1. 在 `layout.html` 定義了這個版型的名稱 **layout** `{{ define "layout" }}`，也就是程式碼 `tmpl.ExecuteTemplate(w, "layout", data)` 中的 `"layout"`。

    1. 在 include 子版型的語法中，eg: `{{ template "navbar" . }}`，有 **`.`**，是指由 `ExecuteTemplate` 傳入的資料。在 ["text/template"](https://golang.org/pkg/text/template/#pkg-index) 有詳細的說明。

1. `nav.html`: Navigation bar。跟 `layout.html` 一樣，一開頭定義這個版型的名稱 `{{ define "navbar" }}`，也就是 `layout.html` 中 `{{ template "navbar" . }}` 的 `"navbar"`。

    ```html { .line-numbers }
    {{ define "navbar" }}
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <a class="navbar-brand" href="#">{{.Title}}</a>
        <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNavDropdown" aria-controls="navbarNavDropdown" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarNavDropdown">
            <ul class="navbar-nav">
                <li class="nav-item {{ if eq .Nav "home" }}active{{ end }}">
                    <a class="nav-link" href="/">首頁</a>
                </li>
                <li class="nav-item {{ if eq .Nav "add" }}active{{ end }}">
                    <a class="nav-link" href="/add">新增</a>
                </li>
                <li class="nav-item {{ if eq .Nav "all" }}active{{ end }}">
                    <a class="nav-link" href="/all">看全部</a>
                </li>
            </ul>
        </div>
    </nav>
    {{ end }}
    ```

1. `test.html`: 內容的子版型，開頭 `{{ define "content" }}` 與上述相同。

    ```html
    {{ define "content" }}

    <script languate="javascript">
        var pair = {{ .TestStruct }};
        var array = {{ .TestArray }};
        var str1 = "{{ .TestString }}";
        var str2 = "{{ .SimpleString }}";
    </script>

    <p>有特殊字元<br />
        {{ .TestString }} <br />
        <a title='{{ .TestString }}'>{{ .TestString }}</a> <br />
        <a href="/{{ .TestString }}">{{ .TestString }}</a> <br />
        <a href="?q={{ .TestString }}">{{ .TestString }}</a> <br />
        <a onx='f("{{ .TestString }}")'>{{ .TestString }}</a> <br />
        <a onx='f({{ .TestString }})'>{{ .TestString }}</a> <br />
        <a onx='pattern = /{{ .TestString }}/;'>{{.TestString }}</a> <br />
    </p>

    <p>無特殊字元<br />
        {{ .SimpleString }} <br />
        <a title='{{ .SimpleString }}'>{{ .SimpleString }}</a> <br />
        <a href="/{{ .SimpleString }}">{{ .SimpleString }}</a> <br />
        <a href="?q={{ .SimpleString }}">{{ .SimpleString }}</a> <br />
        <a onx='f("{{ .SimpleString }}")'>{{ .SimpleString }}</a> <br />
        <a onx='f({{ .SimpleString }})'>{{ .SimpleString }}</a> <br />
        <a onx='pattern = /{{ .SimpleString }}/;'>{{ .SimpleString }}</a> <br />
    </p>

    <p>Compare<br />
        {{ with . }}
        {{ if eq .Num1 .Num2 }} eq {{ else }} ne {{end}}<br/>
        {{ if ne .Num1 .Num2 }} ne {{ else }} eq {{end}}<br/>
        {{ if lt .Num1 .Num2 }} lt {{ else if gt .Num1 .Num2 }} gt {{ else if le .Num1 .Num2 }} le {{ else }} ge {{end}}
        {{ end }}
    </p>

    <p>Array Index<br />
        {{ index .TestArray 0 }}<br />
        {{ index .TestArray 1 }}<br />
        {{ index .TestArray 2 }}<br />
        len: {{ len .TestArray }}<br />
    </p>

    <p>Map<br />
        abc : {{ index .TestMap "abc" }} <br />
        A : {{ index .TestMap "A" }} <br />
        B : {{ index .TestMap "B" }} <br />
    </p>

    <p>Range Array 1<br />
        {{ range .TestArray}}
            {{.}} <br />
        {{ else }}
            No Data
        {{ end }}
        <br />
    </p>

    <p>Range Array 2<br />
        {{ range $idx, $elm := .TestArray }}
            {{ $idx }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>

    <p>Range Empty<br />
        {{ range .EmptyArray}}
            {{ . }} <br />
        {{ else }}
            No Data
        {{ end }}
        <br />
    </p>

    <p>Range Map<br />
        {{ range $key, $elm := .TestMap }}
            {{ $key }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>

    <p>With Empty<br />
        {{with .EmptyArray}}
            Have Data
        {{else}}
            No Data
        {{end}}
    </p>

    <p>With Int Zero Value<br />
        {{with .ZeroInt}}
            Have Data
        {{else}}
            No Data
        {{end}}
    </p>

    <p>Reference Outside Variable<br />
        {{ range $idx, $elm := .TestArray }}
            {{$.SimpleString}} {{ $idx }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>
    {{ end }}
    ```

## Templates 重點語法說明

Go template engine 會依照版型的內容，自動做 escape。

1. 在 `<script></script>` 的效果

    語法：

    ```html
    <script languate="javascript">
        var pair = {{ .TestStruct }};
        var array = {{ .TestArray }};
        var str1 = "{{ .TestString }}";
        var str2 = "{{ .SimpleString }}";
    </script>
    ```

    結果：

    ```html
    <script languate="javascript">
        var pair = {"A":"foo","B":"boo"};
        var array = ["Hello","World","Test"];
        var str1 = "O\x27Reilly: How are \x3ci\x3eyou\x3c\/i\x3e?";
        var str2 = "中文測試";
    </script>
    ```

    1. 如果 string 內容有需要做 escape 時，go template engine 會自動做，eg: `"O\x27Reilly: How are \x3ci\x3eyou\x3c\/i\x3e?"`。
    1. 如果資料是 struct 或 array，會自動轉成 javascript 的 data type 型態。eg: `var pair = {"A":"foo","B":"boo"};` 及 `var array = ["Hello","World","Test"];`。

1. String 自動 Escape 效果

    語法：

    ```html
    <p>有特殊字元<br />
        {{ .TestString }} <br />
        <a title='{{ .TestString }}'>{{ .TestString }}</a> <br />
        <a href="/{{ .TestString }}">{{ .TestString }}</a> <br />
        <a href="?q={{ .TestString }}">{{ .TestString }}</a> <br />
        <a onx='f("{{ .TestString }}")'>{{ .TestString }}</a> <br />
        <a onx='f({{ .TestString }})'>{{ .TestString }}</a> <br />
        <a onx='pattern = /{{ .TestString }}/;'>{{.TestString }}</a> <br />
    </p>

    <p>無特殊字元<br />
        {{ .SimpleString }} <br />
        <a title='{{ .SimpleString }}'>{{ .SimpleString }}</a> <br />
        <a href="/{{ .SimpleString }}">{{ .SimpleString }}</a> <br />
        <a href="?q={{ .SimpleString }}">{{ .SimpleString }}</a> <br />
        <a onx='f("{{ .SimpleString }}")'>{{ .SimpleString }}</a> <br />
        <a onx='f({{ .SimpleString }})'>{{ .SimpleString }}</a> <br />
        <a onx='pattern = /{{ .SimpleString }}/;'>{{ .SimpleString }}</a> <br />
    </p>
    ```

    結果：

    ```html
    <p>有特殊字元<br />
        O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;? <br />
        <a title='O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?'>O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
        <a href="/O%27Reilly:%20How%20are%20%3ci%3eyou%3c/i%3e?">O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
        <a href="?q=O%27Reilly%3a%20How%20are%20%3ci%3eyou%3c%2fi%3e%3f">O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
        <a onx='f("O\x27Reilly: How are \x3ci\x3eyou\x3c\/i\x3e?")'>O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
        <a onx='f(&#34;O&#39;Reilly: How are \u003ci\u003eyou\u003c/i\u003e?&#34;)'>O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
        <a onx='pattern = /O\x27Reilly: How are \x3ci\x3eyou\x3c\/i\x3e\?/;'>O&#39;Reilly: How are &lt;i&gt;you&lt;/i&gt;?</a> <br />
    </p>

    <p>無特殊字元<br />
        中文測試 <br />
        <a title='中文測試'>中文測試</a> <br />
        <a href="/%e4%b8%ad%e6%96%87%e6%b8%ac%e8%a9%a6">中文測試</a> <br />
        <a href="?q=%e4%b8%ad%e6%96%87%e6%b8%ac%e8%a9%a6">中文測試</a> <br />
        <a onx='f("中文測試")'>中文測試</a> <br />
        <a onx='f(&#34;中文測試&#34;)'>中文測試</a> <br />
        <a onx='pattern = /中文測試/;'>中文測試</a> <br />
    </p>
    ```

    1. Go template 會依據字串在版型內的型別，做相對應的 escape。如：url escape or html escape。

1. Compare and if-else

    語法：

    ```html
    <p>Compare<br />
        {{ with . }}
        {{ if eq .Num1 .Num2 }} eq {{ else }} ne {{end}}<br/>
        {{ if ne .Num1 .Num2 }} ne {{ else }} eq {{end}}<br/>
        {{ if lt .Num1 .Num2 }} lt {{ else if gt .Num1 .Num2 }} gt {{ else if le .Num1 .Num2 }} le {{ else }} ge {{end}}
        {{ end }}
    </p>
    ```

    結果：

    ```html
    <p>Compare<br />

        ne <br/>
        ne <br/>
        lt

    </p>
    ```

1. 讀取 Array 值

    語法：

    ```html
    <p>Array Index<br />
        {{ index .TestArray 0 }}<br />
        {{ index .TestArray 1 }}<br />
        {{ index .TestArray 2 }}<br />
        len: {{ len .TestArray }}<br />
    </p>
    ```

    結果：

    ```html
    <p>Array Index<br />
        Hello<br />
        World<br />
        Test<br />
        len: 3<br />
    </p>
    ```

1. 讀取 Map

    語法：

    ```html
    <p>Map<br />
        abc : {{ index .TestMap "abc" }} <br />
        A : {{ index .TestMap "A" }} <br />
        B : {{ index .TestMap "B" }} <br />
    </p>
    ```

    結果：

    ```html
    <p>Map<br />
        abc : DEF <br />
        A : B <br />
        B :  <br />
    </p>
    ```

1. Array Travel (range-else)

    語法：

    ```html
    <p>Range Array 1<br />
        {{ range .TestArray}}
            {{.}} <br />
        {{ else }}
            No Data
        {{ end }}
        <br />
    </p>

    <p>Range Array 2<br />
        {{ range $idx, $elm := .TestArray }}
            {{ $idx }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>

    <p>Range Empty<br />
        {{ range .EmptyArray}}
            {{ . }} <br />
        {{ else }}
            No Data
        {{ end }}
        <br />
    </p>
    ```

    結果：

    ```html
    <p>Range Array 1<br />

            Hello <br />

            World <br />

            Test <br />

        <br />
    </p>

    <p>Range Array 2<br />

            0 : Hello <br />

            1 : World <br />

            2 : Test <br />

        <br />
    </p>

    <p>Range Empty<br />

            No Data

        <br />
    </p>
    ```

1. Map Travel

    語法：

    ```html
    <p>Range Map<br />
        {{ range $key, $elm := .TestMap }}
            {{ $key }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>
    ```

    結果：

    ```html
    <p>Range Map<br />

            A : B <br />

            abc : DEF <br />

        <br />
    </p>
    ```

1. 確認值是否**不是 zero value**，要特別小心當值是 **zero value**，像 `int` 型別，值又是 **"0"** 時，會判定成沒有值，會進到 `else` 的區塊。

    語法：

    ```html
    <p>With Empty<br />
        {{with .EmptyArray}}
            Have Data
        {{else}}
            No Data
        {{end}}
    </p>

    <p>With Int Zero Value<br />
        {{with .ZeroInt}}
            Have Data
        {{else}}
            No Data
        {{end}}
    </p>
    ```

    結果：

    ```html
    <p>With Empty<br />

            No Data

    </p>

    <p>With Int Zero Value<br />

            No Data

    </p>
    ```

1. 在 `with` or `range` 內，需要使用到外部的變數時，可以用 `$` 來讀取外部值。`$` 是指傳入該版型的資料。以 `test.html` 來說，`$` 代表的是 `MyData.Data`。

    語法：

    ```html
    <p>Reference Outside Variable<br />
        {{ range $idx, $elm := .TestArray }}
            {{$.SimpleString}} {{ $idx }} : {{ $elm }} <br />
        {{else}}
            No Data
        {{end}}
        <br />
    </p>
    ```

    結果：

    ```html
    <p>Reference Outside Variable<br />

            中文測試 0 : Hello <br />

            中文測試 1 : World <br />

            中文測試 2 : Test <br />

        <br />
    </p>
    ```

## Build Web with Gorilla Toolkit

- [mux](https://github.com/gorilla/mux): Mux Router，可以定義更彈性的 routing path。
- [securecookie](https://github.com/gorilla/securecookie): 加密 cookie。
- [schema](https://github.com/gorilla/schema): 將 post form 的資料，轉成 struct。
- [csrf](https://github.com/gorilla/csrf): 避免被 CSRF 攻擊[^csrf]。

[^csrf]: [讓我們來談談 CSRF](https://blog.techbridge.cc/2017/02/25/csrf-introduction/)

將綜合以上與 Gorilla Tool Kit，撰寫登入功能。

### gorilla_web/main.go

```go {.line-numbers}
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "html/template"
    "log"
    "net/http"

    "github.com/gorilla/csrf"
    "github.com/gorilla/mux"
    "github.com/gorilla/schema"
    "github.com/gorilla/securecookie"
)

type member struct {
    Email string `json:"email"`
}

func (m *member) String() string {
    memBytes, err := json.Marshal(m)
    if err != nil {
        return err.Error()
    }
    return string(memBytes)
}

func generateHTML(w http.ResponseWriter, data interface{}, files ...string) {
    var tmp []string
    for _, f := range files {
        tmp = append(tmp, fmt.Sprintf("templates/%s.html", f))
    }

    tmpl := template.Must(template.ParseFiles(tmp...))
    tmpl.ExecuteTemplate(w, "layout", data)
}

func redirect(w http.ResponseWriter, target string) {
    w.Header().Set("Location", target)
    w.WriteHeader(http.StatusFound)
}

func index(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "index")
}

func showLogin(w http.ResponseWriter, r *http.Request) {
    generateHTML(w, csrf.TemplateField(r), "layout", "login")
}

func doLogin(w http.ResponseWriter, r *http.Request) {
    form := struct {
        Email    string `schema:"email"`
        Password string `schema:"password"`
        Token    string `schema:"auth_token"`
    }{}

    r.ParseForm()

    if err := schema.NewDecoder().Decode(&form, r.PostForm); err != nil {
        log.Println("schema decode:", err)
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    mem := &member{
        Email: form.Email,
    }

    tmp, err := secureC.Encode(cookieName, mem)
    if err != nil {
        log.Println("encode secure cookie:", err)
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    cookie := &http.Cookie{
        Name:   cookieName,
        Value:  tmp,
        MaxAge: 0,
        Path:   "/",
    }

    http.SetCookie(w, cookie)
    redirect(w, "/member")
}

func logout(w http.ResponseWriter, r *http.Request) {
    cookie := &http.Cookie{
        Name:   cookieName,
        Value:  "",
        MaxAge: -1,
    }

    http.SetCookie(w, cookie)
    redirect(w, "/")
}

func showRegister(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "show register")
}

func doRegister(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "do resgier")
}

func memberIndex(w http.ResponseWriter, r *http.Request) {
    mem, ok := r.Context().Value(ctxKey(cookieName)).(*member)
    if !ok || mem == nil {
        log.Println(mem, ok)
        redirect(w, "/")
        return
    }

    fmt.Fprintln(w, "member:", mem)
}

func memberShowEdit(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "member show edit")
}

func memberDoEdit(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "member do edit")
}

func memberAuthHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // check cookie
        cookie, err := r.Cookie(cookieName)
        if err != nil {
            w.WriteHeader(http.StatusForbidden)
            return
        }

        value := &member{}
        if err := secureC.Decode(cookieName, cookie.Value, value); err != nil {
            log.Println("decode secure cookie:", err)
            w.WriteHeader(http.StatusInternalServerError)
            return
        }

        newRequest := r.WithContext(context.WithValue(r.Context(), ctxKey(cookieName), value))

        next.ServeHTTP(w, newRequest)
    })
}

type ctxKey string

var (
    secureC *securecookie.SecureCookie
)

const (
    hashKey  = "1234567890123456789012345678901234567890123456789012345678901234"
    blockKey = "0123456789abcdef"

    cookieName = "mytest"
)

func main() {

    secureC = securecookie.New([]byte(hashKey), []byte(blockKey))
    r := mux.NewRouter()

    r.HandleFunc("/", index)
    r.HandleFunc("/login", showLogin).Methods("GET")
    r.HandleFunc("/login", doLogin).Methods("POST")
    r.HandleFunc("/logout", logout)
    r.HandleFunc("/register", showRegister).Methods("GET")
    r.HandleFunc("/register", doRegister).Methods("POST")

    s := r.PathPrefix("/member").Subrouter()
    s.HandleFunc("", memberIndex)
    s.HandleFunc("/edit", memberShowEdit).Methods("GET")
    s.HandleFunc("/edit", memberDoEdit).Methods("POST")

    s.Use(memberAuthHandler)

    CSRF := csrf.Protect(
        []byte(`1234567890abcdefghijklmnopqrstuvwsyz!@#$%^&*()_+~<>?:{}|,./;'[]\`),
        csrf.RequestHeader("X-ATUH-Token"),
        csrf.FieldName("auth_token"),
        csrf.Secure(false),
    )

    log.Fatal(http.ListenAndServe(":8080", CSRF(r)))
}
```

### mux

```go {.line-numbers}
r := mux.NewRouter()

r.HandleFunc("/", index)
r.HandleFunc("/login", showLogin).Methods("GET")
r.HandleFunc("/login", doLogin).Methods("POST")
r.HandleFunc("/logout", logout)
r.HandleFunc("/register", showRegister).Methods("GET")
r.HandleFunc("/register", doRegister).Methods("POST")

s := r.PathPrefix("/member").Subrouter()
s.HandleFunc("", memberIndex)
s.HandleFunc("/edit", memberShowEdit).Methods("GET")
s.HandleFunc("/edit", memberDoEdit).Methods("POST")

s.Use(memberAuthHandler)
```

- `Methods(xxx)`: 限定某種 Http Method。
- `r.PathPrefix(xxx).Subrouter()`: 產生子 router, 方便管理子模組。
- `s.Use(xxx)`: 此 router 下的所有 request 都需要先經過某個 handler 處理，類似 Java Servlet Filter 功能。

### csrf

1. 設定:

    ```go {.line-numbers}
    CSRF := csrf.Protect(
        []byte(`1234567890abcdefghijklmnopqrstuvwsyz!@#$%^&*()_+~<>?:{}|,./;'[]\`),
        csrf.RequestHeader("X-ATUH-Token"),
        csrf.FieldName("auth_token"),
        csrf.Secure(false),
    )

    log.Fatal(http.ListenAndServe(":8080", CSRF(r)))
    ```

1. 用 `csrf.TemplateField(r)` 產生 token 並傳給版型

    ```go {.line-numbers}
    func showLogin(w http.ResponseWriter, r *http.Request) {
        generateHTML(w, csrf.TemplateField(r), "layout", "login")
    }
    ```

1. 將 token 放在 html form 內 (login.html)

    ```html
    {{ define "content" }}
    <form method="post" action="/login">
    {{ . }}
        <div class="form-group row">
        <label for="email" class="col-sm-2 col-form-label">Email:</label>
        <div class="col-sm-10">
            <input type="email" class="form-control" id="email" name="email" required>
        </div>
        </div>
        <div class="form-group row">
            <label for="password" class="col-sm-2 col-form-label">Password</label>
            <div class="col-sm-10">
            <input type="password" class="form-control" id="password" name="password" required>
            </div>
        </div>
        <div class="form-group row">
        <div class="col-sm-10">
            <button type="submit" class="btn btn-primary">Submit</button>
        </div>
        </div>
    </form>
    {{ end }}
    ```

    - 輸出的結果，會在 form 加一個 hidden 的 field，name 為 **auth_token**。

### schema

1. 用 Gorilla schema 處理時 crsf，記得要加一個 token 欄位，可以不處理

    ```go {.line-number}
    form := struct {
        Email    string `schema:"email"`
        Password string `schema:"password"`
        Token    string `schema:"auth_token"`
    }{}

    r.ParseForm()

    err := schema.NewDecoder().Decode(&form, r.PostForm)
    ```

### securecookie

1. Initialize

    ```go {.line-number}
    secureC = securecookie.New([]byte(hashKey), []byte(blockKey))
    ```

    - hashKey: 32 or 64 bytes
    - blockKey: 16 (AES-128), 24 (AES-192), 32 (AES-256) bytes

1. Encode and Set Cookie

    ```go {.line-numbers}
    tmp, err := secureC.Encode(cookieName, mem)
    if err != nil {
        log.Println("encode secure cookie:", err)
        w.WriteHeader(http.StatusInternalServerError)
        return
    }

    cookie := &http.Cookie{
        Name:   cookieName,
        Value:  tmp,
        MaxAge: 0,
        Path:   "/",
    }

    http.SetCookie(w, cookie)
    ```

    - cookieName: cookie name
    - value: 可以是字串，也可以是 struct。

1. Read Cookie and Decode

    ```go {.line-numbers}
    cookie, err := r.Cookie(cookieName)
    if err != nil {
        w.WriteHeader(http.StatusForbidden)
        return
    }

    value := &member{}
    if err := secureC.Decode(cookieName, cookie.Value, value); err != nil {
        log.Println("decode secure cookie:", err)
        w.WriteHeader(http.StatusInternalServerError)
        return
    }
    ```

    - cookieName: cookie name
    - value: 原先寫在 cookie 的值

### MiddlewareFunc

Gorilla mux 提供 MiddlewareFunc，讓所有請求某個 URL 時，必須經過此 func 處理後，才會進入該 URL 的 handler func。作用類似 Java Servlet Filter。通常會在這裏面，作身分驗證，並把用戶的資料，變成 request-scope 的資料。

處理方式：讀 Cookie -> 解析 Cookie 資料 -> 轉換成 Reqeust-Scope 資料 -> Next Handler。在 Go，可以使用 Context 的方式，儲存 request-scope 資料。

```go {.line-numbers}
func memberAuthHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // check cookie
        cookie, err := r.Cookie(cookieName)
        if err != nil {
            w.WriteHeader(http.StatusForbidden)
            return
        }

        value := &member{}
        if err := secureC.Decode(cookieName, cookie.Value, value); err != nil {
            log.Println("decode secure cookie:", err)
            w.WriteHeader(http.StatusInternalServerError)
            return
        }

        newRequest := r.WithContext(context.WithValue(r.Context(), ctxKey(cookieName), value))

        next.ServeHTTP(w, newRequest)
    })
}
```

讀取用戶資料時，就不用再重新讀 Cookie。eg:

```go {.line-numbers}
func memberIndex(w http.ResponseWriter, r *http.Request) {
    mem, ok := r.Context().Value(ctxKey(cookieName)).(*member)
    if !ok || mem == nil {
        log.Println(mem, ok)
        redirect(w, "/")
        return
    }

    fmt.Fprintln(w, "member:", mem)
}
```
