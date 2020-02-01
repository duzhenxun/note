# Jetbrains 系列产品2019.2.*最新破解激活方法

最近jetbrains产品激活码被封的厉害。某宝买来的码现在都已用不了，卖家已不再更新新激活码！说是卖家在 **服刑** ？？？我估计是卖家跑路了，不会再继续更新激活码了！无意中发现网上有人免费提供了一个本地注册的破解文件，
下载地址 https://sn9.us/file/259249-417852471
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146110976-1579146110981.png)

### 下面以golang的IDE举例来学习如何本地文件注册ide

1. 先从官方下载2019版的jetbrains系列的ide.
2019.2版的下载可以从这里找到
[https://www.jetbrains.com/go/download/other.html](https://www.jetbrains.com/go/download/other.html)
安装后第一次运时行选试用30天哦！
如果你已不是第一次安装请删除用户目录下的开关文件夹重新打开会提示是否选试用！
如dds的MAC系统 
```sh
➜  ~ cd /Users/dds/Library/Preferences
➜  Preferences ls |grep Go
GoLand2019.1
➜  Preferences rm -rf GoLand2019.1
```
注：如果你是其它系统找不到它的配置路径，干脆卸载了重新安装算了！

2. 准备好本地注册文件（jetbrains-agent.jar 下载的zip解压出来的文件）
选择 “Help”-> “Edit Custom VM Options...”
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146148585-1579146148588.png)

3. 在弹出来的文件中添写你 jetbrains-agent.jar的本地路径，比如：
**-javaagent:/Users/dds/jetbrains-agent.jar** 
“/Users/dds/jetbrains-agent.jar”就是我的本地文件所在位置。
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146170505-1579146170509.png)

4. 将修改host 0.0.0.0 account.jetbrains.com
5. **重新启动ide**,选择“Help"-> "Register..." ,选择"License server" license server address: http://jetbrains-license-server
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146191348-1579146191350.png)

现在ide应可以使用了吧 ~

对于不差钱的用户应该支持一下jetbrains这家公司，小时候没钱会买高仿阿迪耐克，长大后你不差钱了，还会买高仿的吗？我想这家公司也不会如此愚蠢的破解版途径全部封掉吧。关于有些收费用户举报免费用户的行为我想跟他说：“如果就你一个人用这款IDE，其他人改用微软Visual Studio时，为了团队合作估计你老板也会让你改成Visual Studio 工具”！！

最后奉献上jetbrains公司的联系方式，联系电话／微信咨询
个人自费购买：+86 131 2797 3755
公司采购，商务合作：+86 131 8507 2965
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146209452-1579146209458.png)
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146229112-1579146229117.png)

