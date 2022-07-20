# MI8 导出微信聊天记录

> 参考文章：[小米 6 提取微信聊天记录笔记](https://github.com/Heyxk/notes/blob/master/notes/%E5%B0%8F%E7%B1%B3%E6%89%8B%E6%9C%BA%E6%8F%90%E5%8F%96%E5%BE%AE%E4%BF%A1%E8%81%8A%E5%A4%A9%E8%AE%B0%E5%BD%95%E6%95%B0%E6%8D%AE%E5%BA%93.md)

## 通过手机数据备份提取聊天记录

1. 使用手机数据备份功能，导出微信数据

2. 可以通过文件管理 app 里的远程管理功能，在手机上打开一个 ftp 服务。然后在电脑上访问 ftp 服务，将备份完成的数据`/MIUI/backup/allBackup/...`下载到本地。

3. 使用 zip 工具解压`微信xxx.bak`文件，会得到一个`apps`文件夹。

4. `apps`文件夹下面有 `com.tencent.mm` 文件夹，聊天记录数据库就存在 `apps/com.tencent.mm/r/MicroMsg/` 下，打开文件夹会发现里面有 32 位字符(MD5 值)的文件夹(登录过多个用户的有多个)，打开此文件夹其中 `EnMicroMsg.db` 就是要找的数据库文件。

## 获取数据库密码

找到聊天数据库了，但是目前还不能得到聊天记录，因为这个数据库是 sqlcipher 加密数据库，需要密码才能打开。

数据库密码有很多种生成方式：

1. 手机`IMEI`+`uin`(微信用户 id `userinformation`) 将拼接的字符串[MD5 加密](http://tool.chinaz.com/tools/md5.aspx)取前 7 位

   > 如`IMEI`为`123456`，`uin`为`abc`，则拼接后的字符串为`123456abc` 将此字符串用[MD5 加密](http://tool.chinaz.com/tools/md5.aspx)(32 位)后
   >
   > 为`df10ef8509dc176d733d59549e7dbfaf` 那么前 7 位`df10ef8` 就是数据库的密码，由于有的手机是双卡，有多个`IMEI`，或者当手机获取不到`IMEI`时会用默认字符串`1234567890ABCDEF`来代替，由于种种原因，并不是所有人都能得出正确的密码，此时我们可以换一种方法。

2. 反序列化`CompatibleInfo.cfg`和`systemInfo.cfg`

   > 不管是否有多个`IMEI` ，或者是微信客户端没有获取到`IMEI`，而使用默认字符串代替，微信客户端都会将使用的信息保存在`MicroMsg`文件夹下面的`CompatibleInfo.cfg`和`systemInfo.cfg`文件中，可以通过这两个文件来得到正确的密码，但是这两个文件需要处理才能看到信息。

3. 使用 hook 方式得到数据库的密码，这个方法最有效[参考](https://blog.csdn.net/qq_24280381/article/details/73521836)

4. 暴力破解

### 使用反序列化的方式获得密码

首先将 `CompatibleInfo.cfg` 和 `systemInfo.cfg` 以及`EnMicroMsg.db` 复制出来。

使用项目里的 [IMEI.java](./IMEI.java) 文件，将三个文件放在相同目录下

终端运行

```bash
javac IMEI.java
java IMEI systemInfo.cfg CompatibleInfo.cfg
```

运行完成后就会得到密码

```shell
root@ubuntu:~/wechat-export# java IMEI systemInfo.cfg  CompatibleInfo.cfg
The IMEI is : 863976042022274
The uin is : 2017013413
The Key is : *******  # 这将会是你的数据库key密码
```

## 解密数据库

Linux 用户可以使用`sqlcipher`来解密：

```bash
sudo apt-get update
sudo apt-get install sqlcipher
sqlcipher EnMicroMsg.db 'PRAGMA key = "你的key密码"; PRAGMA cipher_use_hmac = off; PRAGMA kdf_iter = 4000; ATTACH DATABASE "decrypted_database.db" AS decrypted_database KEY "";SELECT sqlcipher_export("decrypted_database");DETACH DATABASE decrypted_database;'
```

执行上面的命令之后会得到一个解密后的数据库`decrypted_database.db`，可以使用数据库软件查看

---

得到数据库之后可以分析一下你的聊天记录了~~~
