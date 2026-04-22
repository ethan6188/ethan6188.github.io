# ethan6188.github.io

个人博客：**Hugo** + **PaperMod**，部署在 **GitHub Pages**（`main` 推送后由 Actions 自动构建发布）。


|         | 地址                                                                 |
| ------- | ------------------------------------------------------------------ |
| 中文（默认）  | [https://ethan6188.github.io/](https://ethan6188.github.io/)       |
| English | [https://ethan6188.github.io/en/](https://ethan6188.github.io/en/) |


---

## 本地运行

需要安装 [Hugo extended](https://gohugo.io/installation/) 与 Git。

```bash
git clone --recurse-submodules https://github.com/ethan6188/ethan6188.github.io.git
cd ethan6188.github.io
git submodule update --init --recursive   # 若 clone 时漏了 submodule
hugo server -D
```

浏览器打开终端里提示的地址（一般为 [http://localhost:1313](http://localhost:1313)）。`-D` 会包含草稿。

---

## 写文章与发布

新建、双语命名、标签约定等**维护规范**写在仓库根目录的 **`AGENTS.md`**（给工具与协作者对齐用）。

日常发布流程：

```bash
# 编辑 content/ 下对应 md 后
git add .
git commit -m "post: 简述本次改动"
git push
```

推送 `main` 后，等待 GitHub Actions 完成，站点会在约一分钟内更新。

---

## 主题升级

PaperMod 以 submodule 引入：

```bash
git submodule update --remote themes/PaperMod
git add themes/PaperMod
git commit -m "chore: bump PaperMod"
git push
```

主题仓库：[https://github.com/adityatelange/hugo-PaperMod](https://github.com/adityatelange/hugo-PaperMod)
