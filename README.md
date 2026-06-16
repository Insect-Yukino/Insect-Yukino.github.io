# 博客维护说明

这个项目使用 `Hugo` 构建，主题为 `ananke`，但首页、分类页、文章页都已经做了自定义模板。

本文档主要说明以下几件事：

1. 如何新增分类
2. 如何新增文章
3. 如何新增或替换首页分类图片
4. 如何在文章正文中插入图片
5. 如何新增或替换背景图片
6. 常用运行命令

文章排版与 Markdown 写作规范请看：
[文章写作规范.md](d:/Project/blog/文章写作规范.md)

## 目录结构

项目里和内容维护最相关的目录如下：

```text
content/                         文章内容
archetypes/default.md            新文章默认模板
layouts/index.html               首页模板
layouts/_default/list.html       分类索引页模板
layouts/_default/single.html     文章详情页模板
static/images/category/          首页分类卡图片
static/images/post-avatar/       分类索引页文章卡头像
static/images/background-image/  分类索引页、文章详情页背景图
static/images/                   首页首屏背景图、文章正文图片等
```

## 常用命令

在项目根目录运行：

```powershell
hugo server
```

本地预览地址：

```text
http://localhost:1313/
```

只生成静态文件：

```powershell
hugo
```

## 一、新增分类

### 1. 创建分类目录

在 `content/` 下新建一个目录，目录名就是分类 key。

例如新增 `ComputerOrganization`：

```text
content/ComputerOrganization/
```

然后把这个分类下的文章都放在这个目录里。

### 2. 新建该分类下的第一篇文章

例如：

```text
content/ComputerOrganization/计算机组成原理.md
```

只有当这个目录里至少有一篇文章时，首页和分类页里才会出现这个分类。

### 3. 如果希望首页分类顺序固定，修改 `layouts/index.html`

首页分类不是完全自动排序的，而是由下面 3 组变量控制：

文件：
[layouts/index.html](d:/Project/blog/layouts/index.html)

需要关注的变量：

```gohtml
{{ $preferredKeys := slice ... }}
{{ $preferredImageKeys := slice ... }}
{{ $preferredSections := slice ... }}
```

它们的含义分别是：

1. `$preferredKeys`
   控制首页分类的显示顺序。

2. `$preferredSections`
   控制分类 key 和显示名称的映射。

3. `$preferredImageKeys`
   控制首页分类卡使用哪一张分类图片。
   注意：当前项目里，首页分类文字顺序和图片顺序是分开控制的。

### 4. 新增分类时的标准做法

假设新增分类 `ComputerOrganization`，推荐这样改：

在 `$preferredKeys` 中加入：

```gohtml
"computerorganization"
```

在 `$preferredSections` 中加入：

```gohtml
(dict "key" "computerorganization" "name" "ComputerOrganization")
```

如果你希望首页图片也为这个分类单独安排一张图，再根据需要修改 `$preferredImageKeys`。

### 5. 如果不改首页顺序数组会怎样

如果只创建 `content/ComputerOrganization/` 并写文章，但不修改 `layouts/index.html`：

1. 这个分类仍然会出现
2. 但它会被追加到首页分类列表最后面
3. 分类显示名称会取目录名

## 二、新增文章

### 方式 1：直接手动新建

例如在 `JavaSE` 分类下新增文章：

```text
content/JavaSE/Java异常机制.md
```

文章文件开头使用 front matter，例如：

```toml
+++
date = '2026-06-01T20:00:00+08:00'
draft = false
title = 'Java异常机制'
+++
```

然后下面直接写 Markdown 正文。

### 方式 2：使用 Hugo 命令生成

```powershell
hugo new JavaSE/Java异常机制.md
```

这个命令会使用：
[archetypes/default.md](d:/Project/blog/archetypes/default.md)

生成默认头部模板。

### 文章列表页排序规则

分类页文章列表目前使用的是：

```gohtml
{{ range .RegularPages.ByDate.Reverse }}
```

也就是：

1. 按 `date` 排序
2. 时间新的在前面

所以如果你想控制同一分类中文章出现顺序，改文章的 `date` 即可。

## 三、新增或替换首页分类图片

首页分类卡图片目录：

```text
static/images/category/
```

当前已有的文件名示例：

```text
algorithm.jpg
elasticsearch.jpg
git.jpg
javase.jpg
juc.jpg
linux.jpg
mysql.jpg
redis.jpg
spring.jpg
springboot.jpg
```

### 当前项目的首页分类图片规则

首页图片不是简单按分类 key 自动一一对应，而是由：

[layouts/index.html](d:/Project/blog/layouts/index.html)

中的这一组控制：

```gohtml
{{ $preferredImageKeys := slice ... }}
```

例如列表第 1 个分类卡，使用 `$preferredImageKeys` 中第 1 个 key 对应的图片。

所以如果你想修改首页分类图片顺序，需要改的是：

```gohtml
$preferredImageKeys
```

不是只改 `content/` 目录名。

### 新增一张分类图片的操作步骤

1. 把图片放到：

```text
static/images/category/
```

2. 命名为：

```text
<image-key>.jpg
```

例如：

```text
static/images/category/computerorganization.jpg
```

3. 如果希望首页某个分类卡使用它，就把这个 key 写进：

```gohtml
$preferredImageKeys
```

### 如果分类图片缺失会怎样

如果首页分类卡找不到对应图片，会自动显示一个字母兜底头像，不会报错。

## 四、文章图片怎么加

这里分两种情况。

### 1. 文章正文里的图片

这是最常见的“文章图片”。

推荐做法：

1. 在 `static/images/` 下自建一个更清晰的目录
2. 然后在 Markdown 里直接引用

例如：

```text
static/images/posts/javase/exception-flow.png
```

Markdown 写法：

```md
![异常流程图](/images/posts/javase/exception-flow.png)
```

因为 `static/` 下的文件会直接映射到站点根路径，所以：

```text
static/images/posts/javase/exception-flow.png
```

在文章里就写成：

```text
/images/posts/javase/exception-flow.png
```

### 2. 分类索引页里每篇文章前面的头像

这不是正文图片，而是分类页文章列表里的小头像。

目录：

```text
static/images/post-avatar/
```

当前模板文件：
[layouts/_default/list.html](d:/Project/blog/layouts/_default/list.html)

当前逻辑是：

1. 同一分类页里，文章会按顺序使用 `avatar1.jpg` 到 `avatar8.jpg`
2. 超过 8 篇后，会自动显示编号兜底头像

也就是说：

1. 如果你想替换这批头像，直接替换 `static/images/post-avatar/` 里的同名文件
2. 如果你想增加头像数量，需要同时修改 [layouts/_default/list.html](d:/Project/blog/layouts/_default/list.html) 里的 `$avatarFiles`

## 五、背景图片怎么加

这个项目有 3 类背景图，不是同一个地方。

### 1. 首页首屏背景图

首页模板文件：
[layouts/index.html](d:/Project/blog/layouts/index.html)

当前首页首屏默认图：

```gohtml
background-image: url("/images/cover1.jpg");
```

点击首页首屏时，会在下面这组图片中切换：

```js
const images = [
    "/images/cover1.jpg",
    "/images/cover2.jpg",
    "/images/cover3.jpg",
    "/images/cover4.jpg",
    "/images/cover5.jpg",
    "/images/cover6.jpg"
];
```

所以如果你要新增首页首屏图：

1. 把图片放到：

```text
static/images/
```

2. 命名为：

```text
cover7.jpg
```

3. 再把它追加到 [layouts/index.html](d:/Project/blog/layouts/index.html) 的 `images` 数组里

### 2. 分类索引页背景图

分类页模板：
[layouts/_default/list.html](d:/Project/blog/layouts/_default/list.html)

背景图目录：

```text
static/images/background-image/
```

当前分类页会从下面这些图片里随机取一张：

```js
"/images/background-image/background-img1.jpg"
...
"/images/background-image/background-img10.jpg"
```

如果要新增：

1. 把图片放到：

```text
static/images/background-image/
```

2. 例如命名为：

```text
background-img11.jpg
```

3. 把它加到 [layouts/_default/list.html](d:/Project/blog/layouts/_default/list.html) 的 `images` 数组里

### 3. 文章详情页背景图

文章页模板：
[layouts/_default/single.html](d:/Project/blog/layouts/_default/single.html)

它和分类索引页共用同一套背景图目录：

```text
static/images/background-image/
```

如果你新增了背景图，也要同步把它加入：

[layouts/_default/single.html](d:/Project/blog/layouts/_default/single.html)

里的 `imgs` 数组，否则文章页不会随机到新图片。

## 六、推荐维护流程

### 新增一篇已有分类的文章

1. 在对应 `content/<分类>/` 下新建 `.md`
2. 写好 `title`、`date`、`draft`
3. 如果正文有图，把图放到 `static/images/posts/...`
4. 在 Markdown 里用 `/images/...` 引用
5. 运行 `hugo server` 预览

### 新增一个全新的分类

1. 新建 `content/<分类>/`
2. 至少写一篇文章
3. 修改 [layouts/index.html](d:/Project/blog/layouts/index.html) 中：
   `preferredKeys`
4. 修改 [layouts/index.html](d:/Project/blog/layouts/index.html) 中：
   `preferredSections`
5. 如有需要，修改 [layouts/index.html](d:/Project/blog/layouts/index.html) 中：
   `preferredImageKeys`
6. 准备分类图片到 `static/images/category/`
7. 运行 `hugo server` 检查首页和分类页

### 新增背景图

1. 首页首屏图：
   放到 `static/images/`，并修改 [layouts/index.html](d:/Project/blog/layouts/index.html)
2. 分类页 / 文章页背景图：
   放到 `static/images/background-image/`，并同时修改 [layouts/_default/list.html](d:/Project/blog/layouts/_default/list.html) 和 [layouts/_default/single.html](d:/Project/blog/layouts/_default/single.html)

## 七、容易踩坑的地方

1. `content/` 下只有目录没有文章，这个分类不会正常显示。
2. 首页分类顺序不是纯自动的，想固定顺序必须改 [layouts/index.html](d:/Project/blog/layouts/index.html)。
3. 首页分类图片顺序和分类文字顺序目前是分离控制的，改分类顺序时不要忘了看 `$preferredImageKeys`。
4. 文章正文图片必须放在 `static/` 下，Markdown 里使用站点路径 `/images/...` 引用。
5. 新增背景图后，如果只改一个模板，另一个页面类型可能随机不到新图。
6. 分类页文章卡头像不是正文图片，它来自 `static/images/post-avatar/`。

## 八、修改后如何检查

每次修改完建议运行：

```powershell
hugo server
```

重点检查：

1. 首页分类顺序是否符合预期
2. 首页分类图片是否符合预期
3. 分类页文章排序是否正确
4. 新图片路径是否能正常打开
5. 新背景图是否能被随机到
