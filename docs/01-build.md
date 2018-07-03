# 构建项目

## 下载源码

[Github地址](https://github.com/gohugoio/hugo/)

```
git clone https://github.com/gohugoio/hugo.git
```

## 下载依赖

hugo官方使用了 [Dep](github.com/golang/dep/cmd/dep) 来记录第三方包，
使用 [Mage](github.com/magefile/mage) 自动安装依赖。

```bash
go get github.com/magefile/mage
go get -d github.com/gohugoio/hugo
cd ${GOPATH:-$HOME/go}/src/github.com/gohugoio/hugo
mage vendor
mage install
```

自豪的用中文，当然也可能遇到拦路虎。

另一种办法是自己使用Glide来下载依赖。

`glide.yaml`

```bash
glide init
glide install
```

且，需要配合 glide mirror

- [glide mirror](https://studygolang.com/articles/9278)
- [国人改版Glide, 支持 base/sub repository](https://github.com/xkeyideal/glide)

## 构建

```
cd $GOPATH/src/github.com/gohugo/hugo
go build .
```
