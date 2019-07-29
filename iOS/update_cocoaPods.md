
# 更新gem和cocoapods

### 更新gem

`sudo gem update --system`

### 查看现在的源

`gem sources -l` 可以获取到现在的源

### 移除现在的源

假设现在的源是：https://ruby.taobao.org/

`sudo gem sources --remove https://ruby.taobao.org/`

### 添加新源

`sudo gem source -a http://rubygems-china.oss.aliyuncs.com`

### 安装或者升级cocoapods

`sudo gem install -n /usr/local/bin cocoapods`

### 查看当前cocoapods版本

`pod --version`

### 移除cocoapods

`gem list`
`sudo gem uninstall cocoapods`

