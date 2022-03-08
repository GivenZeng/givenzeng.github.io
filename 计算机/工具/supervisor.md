通过supervisor可以方便运行、监控、控制多个进程
## 安装
```
1：easy_install 安装：
easy_install supervisor
2：pip 安装：
pip install supervisor
3：Debian / Ubuntu可以直接通过apt安装：
apt-get install supervisor
```
## 配置
通过apt-get install安装后，supervisor的配置文件在：/etc/supervisor/supervisord.conf

supervisor的配置文件默认是不全的，不过在大部分默认的情况下，上面说的基本功能已经满足。而其管理的子进程配置文件在：/etc/supervisor/conf.d/\*.conf


## 项目配置
```
#项目名
[program:blog]
#脚本目录
directory=/opt/bin
#脚本执行命令
command=/usr/bin/python /opt/bin/test.py
#supervisor启动的时候是否随着同时启动，默认True
autostart=true
#当程序exit的时候，这个program不会自动重启,默认unexpected
#设置子进程挂掉后自动重启的情况，有三个选项，false,unexpected和true。如果为false的时候，无论什么情况下，都不会被重新启动，如果为unexpected，只有当进程的退出码不在下面的exitcodes里面定义的
autorestart=false
#这个选项是子进程启动多少秒之后，此时状态如果是running，则我们认为启动成功了。默认值为1
startsecs=1
#日志输出
stderr_logfile=/tmp/blog_stderr.log
stdout_logfile=/tmp/blog_stdout.log
#脚本运行的用户身份
user = zhoujy
#把 stderr 重定向到 stdout，默认 false
redirect_stderr = true
#stdout 日志文件大小，默认 50MB
stdout_logfile_maxbytes = 20M
#stdout 日志文件备份数
stdout_logfile_backups = 20
```

文档介绍
```
;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)
```


## 命令
```
//重启suerpvisord
service supervisor start/stop/restart  
//重新加载项目配置文件，重启所有程序
supervisorctl reload
//查看当前子进程状态
supervisorctl status
//控制项目
supervisorctl start/stop/restart app_name
//使用tail输出项目的标注输出
supervisorctl tail app_name stdout
//更新配置文件,如果有新的配置，就会启动
supervisorctl update
//停止所有进程
supervisorctl stop/start/restart all

supervisorctl help
```

## 坑
### 环境变量问题产生spawn error
如：
```
[program:goblog]
#脚本目录
environment = GOPATH ="/home/zjw/go"

#脚本执行命令
command=/usr/local/go/bin/go run /home/zjw/go/src/goblog/app/server.go
#supervisor启动的时候是否随着同时启动，默认True
autostart=false
```
如果缺少environment,会发生错误： cannot find package "goblog/controllers" in any of:/usr/local/go/src/goblog/controllers (from $GOROOT)

### 重新启动的时候发生端口已被使用
原因：当你stop的时候，并没有真正将你的应用杀死，因此端口仍然是在使用中,[参考](https://coderwall.com/p/4tcw7w/setting-supervisor-to-really-stop-django-runserver)

解决方法：
```
[program:some_django]
command=python manage.py runserver
directory=/dir/to/app
#！！！！！！！！！！！！！！！！重点
stopasgroup=true   
```
