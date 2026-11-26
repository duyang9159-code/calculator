# PROJECT

## 项目背景

这是一个面向实验室日常使用的单页网页计算器，部署在 GitHub Pages。页面整合细胞计数、同源重组、铺板接种、转染、溶液配制、PCR、病毒滴度与 MOI 等计算模块。

## 项目目标

- 提供可直接在浏览器和微信中使用的实验计算工具。
- 让不同实验方法使用各自匹配的输入参数、公式和结果展示。
- 保留本地输入记忆与历史记录，减少重复录入。
- 通过 GitHub Pages 持续发布更新。

## 线上信息

- 项目仓库：[duyang9159-code/calculator](https://github.com/duyang9159-code/calculator)
- 网站：[https://duyang9159-code.github.io/calculator/](https://duyang9159-code.github.io/calculator/)
- 微信验证文件：[d359b035f4dab73de0763f9a679d1960.txt](./d359b035f4dab73de0763f9a679d1960.txt)
- 验证文件内容：`fef3e3ee436f9b0c6a5571b8b743b67b2751525d`

## 技术结构

- 主页面和全部前端逻辑集中在 `index.html`。
- 使用原生 HTML、CSS 和 JavaScript，无构建步骤。
- 输入记忆和历史记录使用浏览器 `localStorage`。
- GitHub Pages 从 `main` 分支发布。

## 当前关键约定

- 同源重组最多 5 个实验组，默认 1 组。片段数量全组共用，每组其他参数独立。
- 悬液总细胞量使用自适应科学计数法。
- 病毒滴度有 12 排梯度，复孔数默认 3，最大 8。
- 滴度模块按"计算方法"切换两套独立输入区:TCID50(Reed–Muench)填每排阳性孔数;TU 填每排病变细胞总数。两模式输入分别保存,切换不丢失。
- 微信验证文件必须保留在仓库根目录。
- 微信验证文件必须保留在仓库根目录。

## 当前主要限制

无明显阻塞项。待办与回归检查见 `TODO.md`。
