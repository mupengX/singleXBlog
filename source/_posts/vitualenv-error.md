title: python vitrualenv setuptools pip wheel failed with error code 1的问题

date: 2017-12-03 23:46:10

categories: 怪小兽的日常

tags:

---

在用vituralenv新建Python虚拟环境时遇到如下报错信息：
`setuptools pip wheel failed with error code 1`
看了一下virtualenv的版本是15.1.0。网上查了一下，发现遇到该问题的大部分版本都是V1.11，卸载重新用pip安装还是报同样的错，最后用easy_install安装V1.10.1的virtualenv：`easy_install "virtualenv<1.11"`问题得以解决。