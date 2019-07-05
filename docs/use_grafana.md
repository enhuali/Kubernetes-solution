# grafana使用

# 后台修改用户密码

[官网介绍安装插件进行修改](https://grafana.com/docs/administration/cli/#reset-admin-password)，我们这里看一下直接使用修改数据库进行修改密码

1. 在存储grafana数据目录下查找grafana.db文件，此文件在pvc的根目录下，也可以通过find查找

   ```
   cd grafana-pvc-dir
   
   find ./ -name "grafana.db"
   ```

   

2. 使用sqlite3加载数据库文件

   ```
   sqlite3 grafana.db
   #.tables查看有那些表
   .tables
   #select查看表里面的内容
   select * from user;
   #使用update更新密码
   update user set password = '59acf18b94d7eb0694c61e60ce44c110c7a683ac6a8f09580d626f90f4a242000746579358d77dd9e570e83fa24faa88a8a6', salt = 'F3FAxVm33R' where login = 'admin';
   #修改完成后退出
   .exit
   ```

   此处修改的密码为admin，可以根据以有用户的密码进行修改然后在web端修改成个人密码