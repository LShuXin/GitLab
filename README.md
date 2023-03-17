# GitLab
搭建本地GitLab服务
1. 安装 docker
下载 docker，地址：https://docs.docker.com/docker-for-mac/install/
首先注册 docker 账号,登陆后，即可下载 docker。建议使用迅雷等工具下载，地址：https://download.docker.com/mac/stable/Docker.dmg 下载安装完毕，登录账号
2. 下载镜像 
使用命令行，拉取镜像
```
docker pull gitlab/gitlab-ce
```
会因为超时而报错
```
Using default tag: latest
Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout
```
使用国内镜像
```
Docker -> Preferences -> Daemon，添加镜像地址， Apply & Restart
```
等待片刻，docker 重新 running 的时候，再次执行命令
```
docker pull gitlab/gitlab-ce
```
过程：
```
Using default tag: latest
latest: Pulling from gitlab/gitlab-ce
e80174c8b43b: Pull complete
d1072db285cc: Pull complete
858453671e67: Pull complete
3d07b1124f98: Pull complete
1abbbf4783f5: Pull complete
38a43d00563b: Pull complete
8bbea5a60f40: Pull complete
176bd574f7c7: Pull complete
a8646c9c80ee: Pull complete
089fe821c806: Pull complete
Digest: sha256:88f1bcc39aa9917ac4b19022af441b64265d50e1f0c0fa2616d29a2cb82fb41a
Status: Downloaded newer image for gitlab/gitlab-ce:latest
docker.io/gitlab/gitlab-ce:latest
```
仅仅使用了 7 分钟，就拉取完毕了

3. 运行 gitlab 实例
```
  sudo docker run -d \
    --hostname gitlab.lsx.com \
    --name gitlab \
    --restart always \
    --publish 30001:22 --publish 30000:80 --publish 30002:443 \
    --volume $HOME/gitlab/data:/var/opt/gitlab \
    --volume $HOME/gitlab/logs:/var/log/gitlab \
    --volume $HOME/gitlab/config:/etc/gitlab \
    gitlab/gitlab-ce
```

其中 volume 选项将 gitlab 的目录挂载为用户当地目录，以免容器在停止或被删除的时候丢失数据。publish 选项将宿主机器的 30000、30001 和 30002 映射为容器的 80(http)、22(ssh)和 443(https)端口。

执行完后，输入用户密码，在 home 目录会创建 gitlab 目录
可以下载一个 docker 的可视化工具 Kiteatic。Kiteatic 下载地址[1]

4. 配置 gitlab 实例
```
# 配置文件位置
$HOME/GitLab/gitlab.rb
```
配置访问地址: 

将external_url修改为GitLab服务器的访问地址：
```
external_url 'http://localhost:30000'
```

由于定义的 url 中有端口号，需要将 nginx 监听的端口号改回 80，否则 nginx 将监听容器的 30000 端口，造成 GitLab 无法使用：
```
nginx['listen_port'] = 80
```

配置 ssh 协议所使用的访问地址和端口
```
gitlab_rails['gitlab_ssh_host'] = "localhost"
gitlab_rails['gitlab_shell_ssh_port'] =30001
```

配置邮箱
```
gitlab_rails['gitlab_email_from'] = "xxxx@qq.com”
gitlab_rails['gitlab_email_reply_to'] = ‘xxxx@qq.com'

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "xxxx@qq.com"
# 此处密码应该为客户端授权码，而不是登录密码
gitlab_rails['smtp_password'] = "xxxxpassword"
gitlab_rails['smtp_domain'] = "qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] =true

gitlab_rails['smtp_openssl_verify_mode'] = "peer"
```

注意：

以上设置的端口号 465 是 SSL 协议端口号，非 SSL 协议端口号是 25。此处填写的密码应该是客户端授权码，而不是邮箱的登录密码，如果设置错误，会导致发送邮件失败

用命令 ```docker restart gitlab``` 重启 GitLab，或者在容器中执行命令 ```gitlab-ctl reconfigure``` 重新配置 gitlab。

查看日志
# 实时查看docker容器日志
```
$ sudo docker logs -f -t --tail 行数 容器名
```
5. 测试 

由于之前已经配置了端口映射, 打开浏览器输入http://localhost:30000/，就可以看到登录界面

密码至少要 8 位

至此，安装搭建 git 服务器基本完成。更多相关文档，请查看https://docs.gitlab.com/omnibus/README.html

7. 注意事项
   
- Whoops, GitLab is taking too much time to respond 502 错误： 第一次运行出现这个问题，一段时间之后消失，暂时未查明白原因
- 服务跑起来之后要先设置管理员账号并登陆，而且要修改管理员邮箱，否则后面新用户注册的时候注册的邮件无法发送给管理员，新注册的账号也无法登陆，设置管理员密码的方式如下：
```
cd $HOME/gitlab/bin
sudo gitlab-rails console production # 等一会儿，看到正常输出之后可以进行下一步
u.password='12345678'
u.password_confirmation='12345678'
u.save!
```
8. 相关链接
- gitlab的中文社区，包括汉化包：https://gitlab.com/xhang/gitlab
- gitlab官方网站以及包：https://packages.gitlab.com/gitlab

[1] Kiteatic下载地址: https://download.docker.com/kitematic/Kitematic-Mac.zip



## 找回Root密码

  官方文档说明：https://docs.gitlab.com/ee/security/reset_user_password.html

 1.重置root密码之前，需先使用root用户登录到gitlab所在服务器。并且进入gitlab容器中，使用以下命令启动Ruby on Rails控制台。

```
  gitlab-rails console -e production
```

 2.等待控制台加载完毕，有多种找到用户的方法，您可以搜索电子邮件或用户名。

```
  user = User.where(id: 1).first
```

 或者

```
  user = User.find_by(email: 'admin@example.com')
```

  3.现在更改密码。

```
  user.password = '新密码'
  user.password_confirmation = '新密码'
```

 4.注意，必须同时更改密码和password_confirmation才能使其正常工作，最后别忘了保存(user.save)更改。

```
[root@k8s-node2 ~]# docker ps      // 查看所有运行中的容器
CONTAINER ID   IMAGE               COMMAND                  CREATED              STATUS                         PORTS          NAMES
971e942b7a70   gitlab/gitlab-ce    "/assets/wrapper"        About a minute ago   Up About a minute (health: starting)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:222->22/tcp   gitlab                                                                                           

[root@k8s-node2 ~]# docker exec -it gitlab /bin/bash    // 进入gitlab容器中
root@971e942b7a70:/# gitlab-rails console -e production
--------------------------------------------------------------------------------
 Ruby:         ruby 2.7.4p191 (2021-07-07 revision a21a3b7d23) [x86_64-linux]
 GitLab:       14.3.0 (ceec8accb09) FOSS
 GitLab Shell: 13.21.0

 PostgreSQL:   12.7
--------------------------------------------------------------------------------
Loading production environment (Rails 6.1.3.2)
irb(main):001:0> user = User.where(id: 1).first
=> #<User id:1 @root>
irb(main):002:0> user.password = 'admin1234'
=> "admin1234"
irb(main):004:0> user.password_confirmation = 'admin1234'
=> "admin1234"
irb(main):005:0> user.save
Enqueued ActionMailer::MailDeliveryJob (Job ID: 191a2ed7-0caa-4122-bd06-19c32bffc50c) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", {:args=>[#<GlobalID:0x00007f72f7503158 @uri=#<URI::GID gid://gitlab/User/1>>]}
=> true
```

至此，管理员root用户密码重置完毕，重置后的密码为admin1234。