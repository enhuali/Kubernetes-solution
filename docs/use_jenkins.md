# Jenkins使用

[官网](https://jenkins.io/zh/doc/book/managing/security/)

# 使用规范

- 项目命名必须遵循如下约定：

  业务类型-业务名称，比如B端basetools项目，则命名为B-basetools

- 新增业务需由manager创建项目到对应标签下（B端）后，再进行通知运维人员进行上线，比如新加B端job5项目

  ![image-20190610181553821](pics/use_jenkins_01.png)

  新增项目复制之前的模版即可

  ![image-20190610181654826](pics/use_jenkins_02.png)

  

  修改域名配置中的内容，将期望域名进行更替，多个域名时用英文的逗号进行替换

  ![image-20190610182002088](pics/use_jenkins_03.png)
  



## 系统管理

![image-20190610151149628](pics/use_jenkins_04.png)

`插件管理` 插件的增删更新等操作

`系统配置` `全局工具配置` 插件安装后有些需要进行相关的配置

`管理用户` 用户的创建、删除

`安全全局配置` 安全相关，可配置是否允许用户注册/ldap/用户相关权限（人-组-操作）/ssh等

`系统日志` 记录jenkins相关运行日志，便于查看

`读取配置` 当修改后台配置文件后，可重新加载；慎用，一般不要操作

`系统信息` 记录相关工作目录、插件版本等信息

`负载统计` 统计jenkins的负载情况

`Jenkins命令行接口` 使用接口操作jenkins相关，二次开发时需要用到



# 插件

## 权限控制

- `Role-based Authorization Strategy` 
  可以扩展Jenkins的访问控制功能，以支持更加微妙的身份认证和授权方案，插件安装后在"系统管理下"会多一个"Manage and Assign Roles"按钮
  ![image-20190610154741453](pics/use_jenkins_05.png)

![image-20190610182342988](pics/use_jenkins_06.png)

- 全局配置
  `anyone` 此规则仅有read all的权限，所有用户都需要关联此项

  `job manager` 此规则有创建job的权限，当用户为项目manager时进行关联

- 局部配置
  `B builder` 此规则仅有job build权限，一般分配给项目下的普通组员
  `B manager` 此规则有创建job的权限，当用户为项目manager时进行关联，具有job的最高权限

![image-20190610182743244](pics/use_jenkins_07.png)

此图配置用户lienhuab为B端项目的manager，lienhuac仅有发布权限
用户lienhuab

![image-20190610182900389](pics/use_jenkins_08.png)


用户lienhuac

![image-20190610183004566](pics/use_jenkins_09.png)



## 发布任务描述

需要安装插件 [user build vars plugin](http://wiki.jenkins-ci.org/display/JENKINS/Build+User+Vars+Plugin) 内容描述

否则会报错no known implementation of class jenkins.tasks.SimpleBuildWrapper is named BuildUser
 [description setter](https://plugins.jenkins.io/description-setter) 取得变量BUILD_USER，同理可以取得如下值

| **Property**          | **Default**                        |
| --------------------- | ---------------------------------- |
| BUILD_USER            | Full name (first name + last name) |
| BUILD_USER_FIRST_NAME | First name                         |
| BUILD_USER_LAST_NAME  | Last name                          |
| BUILD_USER_ID         | Jenkins user ID                    |
| BUILD_USER_EMAIL      | Email address                      |

- 缺陷：当job是定时执行的时候，获取不到jenkins登录用户名。
  解决方案：可以通过分析job的历史任务，得到没个job的首次执行登录用户名，和末次执行的登录用户名，进行job的归属者。

在pipeline中添加内容

```
						wrap([$class: 'BuildUser']) {
							env.BUILD_USER = sh(returnStdout: true, script: 'echo "$BUILD_USER"').trim()
						}

						currentBuild.description = "本次发布由<strong><span style='color:#E53333;'> " +
								" ${env.BUILD_USER} </span></strong>发起，发布环境为<strong><span style='color:#E53333;'>" +
								" ${params.ENVIRONMENT_TYPE}</span></strong>，任务类型为<strong><span style='color:#E53333;'>" +
								" ${params.TASK_TYPE}</span></strong>"
```



同时需要将"系统"---"全局安全配置"---"标记格式器" 改成"safe html"

![image-20190704190008569](pics/use_jenkins_10.png)

效果图

![image-20190704190053002](pics/use_jenkins_11.png)