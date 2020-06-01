# bilibili_live_foreign_stream_solution
解决b站海外直播限制问题

## 本教程适合有一定Linux基础用户使用

### 1. 条件
1. 国内服务器一台，推荐使用nat vps
2. 本地到国内的中转服务器一台或多台，取决于当前的线路
   例子：本人在欧洲，首先使用了DO的`伦敦`作为最初的中继，然后中转到阿里国际的`香港`，再转到国内移动`nat`，最后从`nat`向`b站`推流
   `Local -> Do -> HK -> CN nat -> bili`
   如果本地到nat的速度足够，情况最优是 `Local -> CN nat -> bili`，这样延迟会很低，像我这种情况多次中转之后延迟最高可以达到10s

### 2. 基本原理
通过`nginx-rtmp`接收推流，并将流转推到下一个服务器
在nat vps上同样通过`nginx-rtmp`接收推流，但是需要通过`ffmpeg`将流推至bili
原因：使用`nginx-rtmp`自带`push`会导致直播间网页卡死(未知原因)，无法加载，只有手机端能正常观看

### 3. 服务器配置

推荐使用`Ubuntu`或者`Debian`

这里按照`Do -> HK -> nat`的2层中转模式进行

#### 1. Do

安装nginx和rtmp模块，如果想使用最新版nginx需要自己编译源文件和模块，这里使用系统自带版本比较快

`apt update && apt install vim nginx libnginx-mod-rtmp -y`

修改配置文件

`vim /etc/nginx/nginx.conf`

添加编辑好的rtmp区信息粘贴进去

```
rtmp {
    server {
            listen 1935; #默认端口无需修改
            chunk_size 128; #最小128，可能会减少延迟时间

            application a1b2c3d4 { #将a1b2c3d4修改为奇怪的文本就行了，以免被别人扫到，可以乱打几个字母数字，然后base64编码一下，比如a1b2c3d4编码后为YTFiMmMzZDQK
                live on;
                push rtmp://live-lhr03.twitch.tv/app/abcdef; #同步推流twitch，可以保存录像(需要在twitch后台设置)。不需要可以注释掉
                push rtmp://HK_IP/a1b2c3d4; #推荐HK和nat使用相同的app名称
            }
    }
}
```

修改完成后`nginx -t`

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

然后重启nginx `service nginx restart`

#### 2. HK

操作同Do

`apt update && apt install vim nginx libnginx-mod-rtmp -y`

`vim /etc/nginx/nginx.conf`

```
rtmp {
    server {
            listen 1935;
            chunk_size 128; 

            application a1b2c3d4 {
                live on;
                push rtmp://nat_IP:port/a1b2c3d4; 
                # 这里注意了，要在ip后面加上端口，如果是有端口映射的nat主机商，你可以在面板处将30000映射到1935(例子)，然后将port改为30000
            }
    }
}
```

同样
修改完成后`nginx -t`

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

然后重启nginx `service nginx restart`

#### 3. nat

这里就需要安装ffmpeg了，注意`apt install`增加了`ffmpeg`
不过通常这些主机商都没有修改apt源，下载会很慢，可以将源修改成ali的源，具体自己查询`系统版本 + 修改ali源/国内源`

`apt update && apt install vim nginx libnginx-mod-rtmp ffmpeg -y`

`vim /etc/nginx/nginx.conf`

```
rtmp {
    server {
            listen 1935; #如果没有端口映射功能，将这里修改为分配的端口，如果有可以不用修改
            chunk_size 128; 

            application a1b2c3d4 {
                live on;
                exec_push ffmpeg -i rtmp://127.0.0.1:修改为上文listen端口/上文app名称/$name -c copy -f flv rtmp://js.live-send.acg.tv/live-js/?streamname=修改为自己直播间key;
            }
    }
}
```
同样 修改完成后`nginx -t`

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
`service nginx restart`

最后在obs里设置推流地址为
`rtmp://Do_IP/App名称`


## 补充

这里的Do HK只是指代服务器，并不是真正的服务器位置
如果你本地到HK这一层的服务器速度够快的话，完全可以只做`HK -> nat`，甚至单`nat`服务器，只要你本地到nat能全天或者直播时间段跑到超过直播码率的上传速度，那就可以这样操作

### 如何保证速度足够直播？
1. 首先确定直播码率，6000码率，大概就是6Mb的上传速度
2. 在服务器上安装`iperf3`测速软件
3. 测试`本地`到`海外服务器`的速度，再测试`海外`到`nat`的速度，如果每个速度都能达到6Mb以上，那就可以稳定直播
4. 还要考虑到时间段问题，国内晚高峰和国内深夜时间测速差别会很大，尽量考虑在自己直播时间段测试，或者跑crontab jio本测一下整天的速度，多测试几天就能看出各个时间段的速度了
