## 创建 ssh 公钥

```bash
# 进入ssh 查看公钥
cat ~/.ssh/id_rsa.pub

# 如果不存在 则需要创建公钥
ssh-keygen -t rsa -C gershonv@163.com
```

复制完公钥后，我们先登陆进服务器。

## 在服务器的 ssh 中添加 authorized_keys

在云服务器中进行以下操作：

```bash
cd ~/.ssh/

ls # 查看是否存在 authorized_keys 文件

vim authorized_keys

# 如果没有的话
vim ~/.ssh/authorized_keys
```

