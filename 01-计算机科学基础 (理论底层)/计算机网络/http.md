```http
POST /file/upload HTTP/1.1                                   ← 请求行
Host: localhost:8080                                          ← 必选头
User-Agent: Mozilla/5.0                                       ← 可选头
Accept: text/html                                             ← 可选头
Cookie: session=abc123                                        ← 可选头
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW  ← 关键头
Content-Length: 54321                                         ← 必选头（有Body时）
Connection: keep-alive                                        ← 可选头
Referer: https://localhost:8080/home                          ← 可选头

------WebKitFormBoundary7MA4YWxkTrZu0gW                       ← 分隔符（boundary）
Content-Disposition: form-data; name="file"; filename="example.jpg"  ← 嵌套头（描述字段）
Content-Type: image/jpeg                                      ← 嵌套头（描述文件类型）

(此处是 example.jpg 的二进制数据，共 54000 字节)              ← 字段值（文件内容）

------WebKitFormBoundary7MA4YWxkTrZu0gW--                      ← 结束分隔符（带 --）
```





## Content-Type：

### 1. 文本类型 (Text)

| Content-Type      | 含义            |
| :---------------- | :-------------- |
| `text/plain`      | 纯文本文件      |
| `text/html`       | HTML 网页内容   |
| `text/css`        | CSS 样式表      |
| `text/javascript` | JavaScript 代码 |
| `text/xml`        | XML 纯文本      |

### 2. 应用类型 (Application) —— API 交互最常用

| Content-Type                        | 含义                                          |
| :---------------------------------- | :-------------------------------------------- |
| `application/json`                  | **JSON 数据格式**（RESTful API 主流）         |
| `application/xml`                   | XML 数据格式                                  |
| `application/x-www-form-urlencoded` | **表单默认提交格式**（key=value&key2=value2） |
| `application/octet-stream`          | **二进制流**（不知道具体类型的文件下载）      |
| `application/pdf`                   | PDF 文件                                      |
| `application/zip`                   | ZIP 压缩包                                    |
| `application/msword`                | Word 文档（.doc）                             |
| `application/vnd.ms-excel`          | Excel 文档（.xls）                            |

### 3. 图像类型 (Image)

| Content-Type    | 含义             |
| :-------------- | :--------------- |
| `image/jpeg`    | JPEG 图片        |
| `image/png`     | PNG 图片         |
| `image/gif`     | GIF 图片         |
| `image/webp`    | WebP 图片        |
| `image/svg+xml` | SVG 矢量图       |
| `image/x-icon`  | 图标文件（.ico） |

### 4. 音视频类型 (Audio/Video)

| Content-Type | 含义      |
| :----------- | :-------- |
| `audio/mpeg` | MP3 音频  |
| `video/mp4`  | MP4 视频  |
| `video/webm` | WebM 视频 |

### 5. 多部分类型 (Multipart) —— 文件上传专用

| Content-Type           | 含义                           |
| :--------------------- | :----------------------------- |
| `multipart/form-data`  | **文件上传**（你例子中的类型） |
| `multipart/mixed`      | 混合多种数据类型               |
| `multipart/byteranges` | 部分内容响应                   |

> **一句话总结 Content-Type**：它的种类就是 **MIME 类型的数量**——标准注册的有数千种，日常开发只需要记住上面这 30 种左右就够用了。



## 二、Content-Disposition：有多少种？

与 `Content-Type` 不同，`Content-Disposition` 的取值**极其有限**。根据 RFC 6266（HTTP 场景的标准），它只有 **2 种标准值**，加上一个扩展用法。

### 标准值（仅 2 种）

| 值               | 含义             | 浏览器行为                                                  |
| :--------------- | :--------------- | :---------------------------------------------------------- |
| **`inline`**     | **内联展示**     | 如果能预览（如图片、PDF），直接在浏览器打开；不能预览则下载 |
| **`attachment`** | **作为附件下载** | 强制弹出“另存为”对话框，不会自动打开                        |

### 扩展值（1 种特殊场景）

| 值              | 出现位置                                 | 含义                                           |
| :-------------- | :--------------------------------------- | :--------------------------------------------- |
| **`form-data`** | **multipart/form-data 的每个子部分内部** | 告诉服务器：这个 part 是一个表单字段，不是附件 |

> ⚠️ **重要**：`form-data` 只能出现在 **multipart 请求体的每个 part 的头部**，不能出现在 HTTP 主响应头里。

### 配套参数（filename / name）

`Content-Disposition` 的值后面可以跟参数，最常见的是：

| 参数        | 示例                                      | 含义                                                     |
| :---------- | :---------------------------------------- | :------------------------------------------------------- |
| `filename`  | `filename="example.jpg"`                  | 建议的下载/保存文件名（不支持中文/特殊字符时的降级方案） |
| `filename*` | `filename*=UTF-8''%E5%9B%BE%E7%89%87.jpg` | 带编码规范的文件名（支持中文和国际字符）                 |
| `name`      | `name="file"`                             | **仅用于 `form-data`**：指定表单字段名                   |

> **一句话总结 Content-Disposition 的种类**：**2 种标准值**（`inline` / `attachment`）+ **1 种特殊扩展值**（`form-data`，仅用于 multipart 内部）。



## 三、对照表：两个 Content-Type 的区别（帮你扫清迷惑）

你最初给的例子中有两个 `Content-Type`，它们**名称相同但层级不同**：

| 特性                 | **外层 Content-Type**               | **内层 Content-Type**                            |
| :------------------- | :---------------------------------- | :----------------------------------------------- |
| **出现位置**         | HTTP 头部（Headers 区）             | `multipart/form-data` 请求体的每个 part 内部     |
| **作用范围**         | 描述**整个请求体**的类型            | 描述**当前这一个 part** 的数据类型               |
| **在你的例子中的值** | `multipart/form-data; boundary=...` | `image/jpeg`                                     |
| **值的种类**         | 所有 MIME 类型（数百种）            | 所有 MIME 类型（数百种）——其实和外层是同一套标准 |

```
┌─────────────────────────────────────────────────────────┐
│ POST /file/upload HTTP/1.1                              │
│ Content-Type: multipart/form-data; boundary=xxx  ← 外层 │
│                                                         │
│ --xxx                                                   │
│ Content-Disposition: form-data; name="file";            │
│ Content-Type: image/jpeg                          ← 内层 │
│                                                         │
│ (图片二进制数据)                                          │
│ --xxx--                                                 │
└─────────────────────────────────────────────────────────┘
```











## 📡 一、HTTP 协议层面：数据是如何组织的

### 浏览器发送的原始 HTTP 请求体

```text
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="username"

张三
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<二进制图片数据....>
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```



### 关键点：`name` 参数

每个 `Content-Disposition` 头部中都有一个 `name` 参数：

- `name="username"` → 这是文本字段
- `name="file"` → 这是文件字段

**`name` 就是前后端的约定暗号！**

------

## 🔍 二、Go 内部如何找出数据

### 第1步：`r.FormFile("file")` 被调用

```go
file, header, err := r.FormFile("file")  // ← 传入 "file"
```



### 第2步：解析 HTTP 请求体

```go
func (r *Request) FormFile(key string) (multipart.File, *multipart.FileHeader, error) {
    // key = "file"
    
    // 解析 multipart 表单
    r.ParseMultipartForm(32 << 20)  // 32MB
    
    // r.MultipartForm.File 是一个 map
    // 结构：map[string][]*FileHeader
    // 例如：map["file"] = [文件1, 文件2...]
    //       map["avatar"] = [文件...]
    
    return r.MultipartForm.File[key][0].Open()
}
```



### 第3步：查找过程（内部实现）

```go
// 简化版内部逻辑
func parseMultipartForm(r *Request) {
    // 1. 创建一个 map 存储文件
    fileMap := make(map[string][]*FileHeader)
    
    // 2. 逐部分扫描 HTTP 请求体
    reader := multipart.NewReader(r.Body, boundary)
    
    for {
        part, err := reader.NextPart()  // 读取下一个部分
        if err == io.EOF {
            break
        }
        
        // 3. 获取这个部分的 name 参数
        // part.Header.Get("Content-Disposition")
        // 内容: "form-data; name=\"file\"; filename=\"photo.jpg\""
        name := extractNameFromDisposition(part.Header)  // 得到 "file"
        
        // 4. 如果是文件类型（有 filename）
        if filename := part.FileName(); filename != "" {
            // 存入 map，key 就是 name
            fileMap[name] = append(fileMap[name], &FileHeader{
                Filename: filename,
                // 保存位置信息，不保存实际内容
            })
        }
    }
    
    r.MultipartForm.File = fileMap
}
```



------

## 📋 三、完整的查找流程图

```text
HTTP 请求体
│
├─ 边界1 ─────────────────────────────┐
│  name="username"                    │
│  数据: "张三"                        │
└─────────────────────────────────────┘
│
├─ 边界2 ─────────────────────────────┐
│  name="file"  filename="photo.jpg"  │ ← 找到 name="file"
│  数据: [二进制图片...]               │
└─────────────────────────────────────┘
│
└─ 边界3 ─────────────────────────────┐
   name="age"                         
   数据: "25"                         
└─────────────────────────────────────┘

                    ↓
            ParseMultipartForm 扫描
                    ↓
        创建 map[string][]*FileHeader
                    ↓
        {
            "username": nil,  // 不是文件，不存入
            "file": [&FileHeader{Filename: "photo.jpg", ...}],
            "age": nil,
        }
                    ↓
            r.FormFile("file")
                    ↓
        r.MultipartForm.File["file"]  ← 通过 key 获取
                    ↓
        返回文件读取器
```



------

## 💡 四、为什么必须传入 `"file"`？

### 原因1：HTTP 协议的设计

HTTP 协议本身就是**键值对**的设计思想：

- 发送方（浏览器）用 `name` 作为键
- 接收方（服务器）用相同的键来获取值

```javascript
// 前端
<input name="file" type="file">  // 键是 "file"
```



```go
// 后端
r.FormFile("file")  // 用相同的键 "file" 获取
```



### 原因2：支持多字段

一个表单可能有多个文件：

```html
<input name="avatar" type="file">
<input name="gallery" type="file" multiple>
```

```go
// 分别获取不同字段的文件
avatarFile, _, _ := r.FormFile("avatar")
galleryFiles := r.MultipartForm.File["gallery"]  // 多个文件
```











## `header` 具体存储的内容

在你的代码中：

go

```
file, header, err := r.FormFile("file")
```



这个 `header` 是 `*multipart.FileHeader` 类型，它存储的是 **multipart 中某一个 part 的头部信息**。

------

## 🔍 一、`header` 包含的具体字段

go

```
type FileHeader struct {
    Filename string            // 文件名，如 "example.jpg"
    Header   textproto.MIMEHeader  // 该 part 的 HTTP 头部
    Size     int64             // 文件大小（字节）
}
```



### 实际例子

假设浏览器发送的 HTTP 请求中，文件部分是这样的：

http

```
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="example.jpg"
Content-Type: image/jpeg
Content-Transfer-Encoding: binary

[文件二进制数据...]
```



那么 `header` 里存储的是：

go

```
header.Filename = "example.jpg"
header.Size = 54321  // 假设文件大小

// header.Header 的内容（textproto.MIMEHeader 类型）
header.Header = {
    "Content-Disposition": []string{"form-data; name=\"file\"; filename=\"example.jpg\""},
    "Content-Type":        []string{"image/jpeg"},
    "Content-Transfer-Encoding": []string{"binary"},
}
```



------

## 📋 二、`header.Header` 可能包含的常见字段

| HTTP 头部字段               | 示例值                                           | 说明                                |
| :-------------------------- | :----------------------------------------------- | :---------------------------------- |
| `Content-Disposition`       | `form-data; name="file"; filename="example.jpg"` | 表单字段信息，包含 name 和 filename |
| `Content-Type`              | `image/jpeg`, `application/pdf`, `text/plain`    | 文件的 MIME 类型                    |
| `Content-Transfer-Encoding` | `binary`, `base64`, `7bit`, `8bit`               | 传输编码方式                        |
| `Content-Length`            | `54321`                                          | 该 part 的数据长度（很少用）        |
| `Content-ID`                | `<part1@example.com>`                            | 可选，用于标识 part                 |
| `Content-Description`       | `A photo of sunset`                              | 可选，描述信息                      |

------

## 🎯 三、`header` 与 `r.Header` 的区别

### `r.Header`（HTTP 请求头）

存储的是**整个 HTTP 请求**的头部信息：

http

```
POST /file/upload HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
Content-Length: 54321
Accept: text/html
Cookie: session=abc123
```



go

```
// r.Header 的内容
r.Header = {
    "Host": []string{"localhost:8080"},
    "User-Agent": []string{"Mozilla/5.0"},
    "Content-Type": []string{"multipart/form-data; boundary=----WebKitFormBoundary"},
    "Content-Length": []string{"54321"},
    "Accept": []string{"text/html"},
    "Cookie": []string{"session=abc123"},
}
```



### `header`（multipart part 头部）

只存储**某个特定 part** 的头部信息：

http

```
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="example.jpg"
Content-Type: image/jpeg
```



go

```
// header.Header 的内容（注意是小写 header，不是 r.Header）
header.Header = {
    "Content-Disposition": []string{"form-data; name=\"file\"; filename=\"example.jpg\""},
    "Content-Type": []string{"image/jpeg"},
}
```



------

## 🔄 四、完整的数据层次结构

text

```
HTTP 请求
│
├─ r.Header (整个请求的头部)
│  ├─ Host: localhost:8080
│  ├─ Content-Type: multipart/form-data; boundary=xxx
│  └─ User-Agent: Mozilla/5.0
│
└─ r.Body (请求体)
   │
   └─ multipart 格式的 body
      │
      ├─ Part 1 (普通字段)
      │  ├─ part.Header
      │  │  └─ Content-Disposition: form-data; name="username"
      │  └─ 数据: "张三"
      │
      └─ Part 2 (文件字段)  ← 这就是你的 file part
         ├─ part.Header  ← 这就是 header.Header 的内容
         │  ├─ Content-Disposition: form-data; name="file"; filename="example.jpg"
         │  └─ Content-Type: image/jpeg
         └─ 数据: [文件二进制内容]  ← 这就是 file.Read() 读到的
```



------

## 💡 五、实际代码演示

go

```
func UploadHandler(w http.ResponseWriter, r *http.Request) {
    file, header, err := r.FormFile("file")
    if err != nil {
        return
    }
    defer file.Close()
    
    // 1. 打印文件元数据
    fmt.Printf("文件名: %s\n", header.Filename)
    fmt.Printf("文件大小: %d 字节\n", header.Size)
    
    // 2. 打印该 part 的所有头部信息
    for key, values := range header.Header {
        for _, value := range values {
            fmt.Printf("Part Header - %s: %s\n", key, value)
        }
    }
    
    // 3. 打印整个 HTTP 请求头
    for key, values := range r.Header {
        for _, value := range values {
            fmt.Printf("Request Header - %s: %s\n", key, value)
        }
    }
    
    // 输出示例：
    // Part Header - Content-Disposition: form-data; name="file"; filename="example.jpg"
    // Part Header - Content-Type: image/jpeg
    // Request Header - Host: localhost:8080
    // Request Header - User-Agent: Mozilla/5.0
    // Request Header - Content-Length: 54321
}
```



------

## 📊 六、对比总结表

| 特性         | `r.Header`                             | `header.Header` (multipart.FileHeader) |
| :----------- | :------------------------------------- | :------------------------------------- |
| **所属层级** | HTTP 请求层                            | Multipart part 层                      |
| **存储内容** | 整个请求的头部                         | 单个文件/字段的头部                    |
| **典型字段** | Host, User-Agent, Cookie, Content-Type | Content-Disposition, Content-Type      |
| **何时可用** | 任何时候                               | 解析 multipart 表单后                  |
| **作用范围** | 整个请求                               | 单个 part                              |
| **访问方式** | `r.Header.Get("key")`                  | `header.Header.Get("key")`             |

------

## 🎯 七、关键要点

1. **`header` 不是整个 HTTP 请求头**，只是 multipart 中一个部分的头部
2. **最重要的信息是 `Filename`**，这是从 `Content-Disposition` 中解析出来的
3. **`Content-Type`** 告诉你文件的 MIME 类型（图片、PDF、文本等）
4. **可以自定义额外字段**，比如 `Content-Description` 来描述这个文件

这个 `header` 让你能够获取上传文件的**元信息**，而不需要读取文件内容本身，非常高效！