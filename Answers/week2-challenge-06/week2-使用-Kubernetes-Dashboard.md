# 挑战：使用 Kubernetes Dashboard

在集群中执行如下命令，避免创建 MySQL Pod 失败：

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
```

点击页面右上方的 CREATE 按钮进行创建：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797083466/wm)

复制 `mysql-rc.yaml` 文件的内容到 `CREATE FROM TEXT INPUT`，然后点击 UPLOAD：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797328506/wm)

等待 mysql Pod 的创建：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797481398/wm)

mysql Pod 创建成功：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797503521/wm)

在 `/home/shiyanlou` 目录下新建 `mysql-svc.yaml` 文件并写入对应的代码，使用 `CREATE FROM FILE` 方式进行创建：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797660919/wm)

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568797730608/wm)

参考 myweb-rc.yaml 文件的内容，使用 `CREATE AN APP` 的方式进行创建，注意相应数据的填写：

- App name：myweb
- Container image：registry-vpc.cn-hangzhou.aliyuncs.com/chenshi-kubernetes/tomcat-app:v1
- Labels：Key 为 app，Value 为 myweb
- Environment variables：
	- Name 为 MYSQL_SERVICE_HOST，Value 为 mysql
	- Name 为 MYSQL_SERVICE_PORT，Value 为 3306

![图片描述](https://doc.shiyanlou.com/courses/1494/600404/a01b94e31e4bfc22e6ffa0490b08d77b-0/wm)

![图片描述](https://doc.shiyanlou.com/courses/1494/600404/7c297f9b61bfedf0d4439623c84b28b0-0/wm)

等待 myweb Pod 创建：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568800082611/wm)

myweb Pod 创建成功：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568800095841/wm)

复制 myweb-svc.yaml 文件的内容到 `CREATE FROM TEXT INPUT`，然后点击 UPLOAD：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568800127258/wm)

查看 myweb Service 创建成功：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568800135678/wm)

最后通过浏览器能够成功访问 `10.192.0.2:30001/demo/index.jsp` 页面：

![图片描述](https://doc.shiyanlou.com/courses/uid600404-20190918-1568800195713/wm)
