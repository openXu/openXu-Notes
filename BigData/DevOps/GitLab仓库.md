# GitLab搭建

ubuntu 1.4
16G

1. 首先安装必须的一些服务

```xml
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates
sudo apt-get install -y postfix
使用左右键和回车键选择确定、取消，弹出列表选项的时候，选择 Internet Site
```

2. 接着信任 GitLab 的 GPG 公钥:

```xml
curl https://packages.gitlab.com/gpg.key 2> /dev/null | sudo apt-key add - &>/dev/null
```

3. 配置镜像路径(由于国外的下载速度过慢，所以配置清华大学镜像的路径)

```xml
vi /etc/apt/sources.list.d/gitlab-ce.list

写入:根据你的版本，选择对于的内容写入/etc/apt/sources.list.d/gitlab-ce.list
	deb https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu trusty main
```
详见[Gitlab Community Edition 镜像使用帮助](https://mirror.tuna.tsinghua.edu.cn/help/gitlab-ce/)

4. 安装 gitlab-ce

```xml
sudo apt-get update
sudo apt-get install gitlab-ce

执行配置
sudo gitlab-ctl reconfigure
启动gitlab
sudo gitlab-ctl start
```

5. 浏览器进行访问

[浏览器进行访问http://192.168.1.193](http://192.168.1.193)
[浏览器进行访问http://192.168.1.193:8899](http://192.168.1.193:8899)

**错误1：Gitlab 403 forbidden**
```xml
错误:
原因:Gitlab使用rack_attack做了并发访问的限制
解决方案:将Gitlab的IP设置为白名单即可
步骤如下:

打开/etc/gitlab/gitlab.rb文件。

查找gitlab_rails['rack_attack_git_basic_auth']关键词。

取消注释

修改ip_whitelist白名单属性，加入Gitlab部署的IP地址。

gitlab_rails['rack_attack_git_basic_auth'] = { 'enabled' => true, 'ip_whitelist' => ["127.0.0.1","Gitlab部署的IP地址192.168.1.193"], 'maxretry' => 300, 'findtime' => 5, 'bantime' => 60 }
配置好后，执行gitlab-ctl reconfigure即可
```

**错误2：502 Whoops, GitLab is taking too much time to respond.**

```xml
修改gitlab的端口和地址
external_url 'http://192.168.1.193:8899'
防火墙开放端口
firewall-cmd --zone=public --add-port=8899/tcp --permanent

将下面这3行打开注释
681line    unicorn['port'] = 8088
776line    postgresql['shared_buffers'] = "256MB"
792line    postgresql['max_connections'] = 200

gitlab-ctl reconfigure
gitlab-ctl restart

```

[从一个git仓库迁移到另外一个git仓库](https://blog.csdn.net/samxx8/article/details/72329002)

6. 设置密码

第一次访问默认会重定向到密码设置页面，账户为root，自己设置密码fpcgitlab

7. 登录使用，跟github差不多

8. 创建用户

root   fpcgitlab
openXu  467856641@qq.com	fpcgitlab

9. 上传项目

首先在网站创建用户组、项目

```xml
git init
git add .
git commit -m "a"
git remote add origin http://192.168.1.193:8899/fpc/gitlabtest.git
git push -u origin master

输入用户名密码
```

# CODO 仓库管理

## 添加仓库

[Access Tokens](http://192.168.1.193:8899/profile/personal_access_tokens)

gitlab右上角点击用户->Settings->左边菜单栏Access Tokens->填写信息创建

[代码仓库设置](http://192.168.1.193:8008/operation_center/codeRepository)

http://192.168.1.193:8899/
AoED2kD8PdDSs4GRWenD

点击刷新地址

## 添加钩子

Git也具有在特定事件发生之前或之后执行特定脚本代码功能（从概念上类比，就与监听事件、触发器之类的东西类似）。Git Hooks就是那些在Git执行特定事件（如commit、push、receive等）后触发运行的脚本。

每一个Git repo下都包含有.git/hoooks这个目录（本地和远程都是--本地Hooks与服务端Hooks），这里面就是放置Hooks的地方。你可以在这个目录下自由定制Hooks的功能，当触发一些Git行为时，相应地Hooks将被执行。把后缀.sample去掉，或者以列表中的名字直接命名，就会把该脚本绑定到特定的Git行为上。

注意：由于Gitlab版本存在CE版本和源码安装版本，全局update钩子配置不同

> GitlabCE版本路径：/opt/gitlab/embedded/service/gitlab-shell/hooks/
> GitLab源码版本路径：/home/git/gitlab-shell/hooks/

[Gitlab api](http://192.168.1.193:8899/help/api/README.md)

[查看所有project id](http://192.168.1.193:8899/api/v4/projects?per_page=500&private_token=AoED2kD8PdDSs4GRWenD)

```JSON
[
    {
        "id":3,
        "description":"法之运gitlab测试",
        "name":"GitlabTest",
        "name_with_namespace":"fpc / GitlabTest",
        "path":"gitlabtest",
        "path_with_namespace":"fpc/gitlabtest",
        "created_at":"2020-06-12T09:44:21.460Z",
        "default_branch":"master",
        "tag_list":[

        ],
        "ssh_url_to_repo":"git@192.168.1.193:fpc/gitlabtest.git",
        "http_url_to_repo":"http://192.168.1.193:8899/fpc/gitlabtest.git",
        "web_url":"http://192.168.1.193:8899/fpc/gitlabtest",
        "readme_url":null,
        "avatar_url":null,
        "star_count":0,
        "forks_count":0,
        "last_activity_at":"2020-06-13T00:10:26.477Z",
        "namespace":{
            "id":3,
            "name":"fpc",
            "path":"fpc",
            "kind":"group",
            "full_path":"fpc",
            "parent_id":null
        },
        "_links":{
            "self":"http://192.168.1.193:8899/api/v4/projects/3",
            "issues":"http://192.168.1.193:8899/api/v4/projects/3/issues",
            "merge_requests":"http://192.168.1.193:8899/api/v4/projects/3/merge_requests",
            "repo_branches":"http://192.168.1.193:8899/api/v4/projects/3/repository/branches",
            "labels":"http://192.168.1.193:8899/api/v4/projects/3/labels",
            "events":"http://192.168.1.193:8899/api/v4/projects/3/events",
            "members":"http://192.168.1.193:8899/api/v4/projects/3/members"
        },
        "archived":false,
        "visibility":"private",
        "resolve_outdated_diff_discussions":false,
        "container_registry_enabled":true,
        "issues_enabled":true,
        "merge_requests_enabled":true,
        "wiki_enabled":true,
        "jobs_enabled":true,
        "snippets_enabled":true,
        "shared_runners_enabled":true,
        "lfs_enabled":true,
        "creator_id":1,
        "import_status":"none",
        "open_issues_count":0,
        "public_jobs":true,
        "ci_config_path":null,
        "shared_with_groups":[

        ],
        "only_allow_merge_if_pipeline_succeeds":false,
        "request_access_enabled":false,
        "only_allow_merge_if_all_discussions_are_resolved":false,
        "printing_merge_request_link_enabled":true,
        "merge_method":"merge",
        "external_authorization_classification_label":null,
        "permissions":{
            "project_access":null,
            "group_access":null
        }
    }
]
```
 
**accept_task_url**: https://192.168.1.193:8899/api/task/other/v1/git/hooks/


























