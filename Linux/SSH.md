#### [免密钥](https://blog.csdn.net/furzoom/article/details/79139570)

1. 生成密钥

   ```shell
   ssh-keygen -t rsa
   ```

2. 将公钥复制到远程机器上

   ```shell
   ssh-copy-id -i ~/.ssh/id_rsa.pub {user_name}@{remote_ip}
   ```

3. 测试

   ```shell
   ssh {user_name}@{remote_ip}
   ```

#### [SCP](https://www.cnblogs.com/zhaofeng555/p/8075279.html)

##### 从选程到本地

```shell
scp [-r] {user_name}@{remote_ip}:{remte_file} {local_file}
```

