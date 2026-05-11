# lazy-supersplat

懒猫微服 lpk 包装：[playcanvas/supersplat](https://github.com/playcanvas/supersplat)

> SuperSplat 是 PlayCanvas 开源的 3D 高斯泼溅 (Gaussian Splat) 编辑器，
> 整个编辑器跑在浏览器里（WebGL + PlayCanvas 引擎），可编辑 `.ply` /
> `.splat` / `.ksplat`：旋转、选区、修剪、球谐 (SH) 编辑、导出、发布。

## 安装

通过懒猫微服「应用商店」搜索 *SuperSplat* 安装。

- 无需任何配置参数 —— 安装即用
- 子域名：`https://supersplat.<your-box>.heiyu.space`

## 使用

1. 在桌面浏览器中打开 `https://supersplat.<your-box>.heiyu.space`
2. 把 `.ply` / `.splat` / `.ksplat` 文件直接拖到画布
3. 旋转、选区编辑、修剪离群点、编辑 SH、导出

## 数据

**服务端只有 nginx + 静态文件，不存任何用户数据。**

上传/编辑/导出全部发生在你的浏览器本地。

## 资源占用

- 服务端：几乎不占资源（一个 nginx 进程）
- 客户端：渲染压力在浏览器一侧 —— 推荐桌面浏览器 + 独立 GPU
- 超大模型（>2GB）建议浏览器加 `--js-flags=--max-old-space-size=8192`

## License 与重打包说明

- 上游项目：[playcanvas/supersplat](https://github.com/playcanvas/supersplat) (MIT)
- 本仓库做的事：仅加一个 Dockerfile + nginx.conf 把上游的 `dist/`
  build 输出托管到 lazycat 子域名 —— 不修改源码、不改前端 LOGO

## 升级

```sh
git subtree pull --prefix=vendor/supersplat \
  https://github.com/playcanvas/supersplat.git main --squash
git apply --check patches/*.patch -p1 --directory=vendor/supersplat
# bump version 后打 tag 触发 release.yml
```
