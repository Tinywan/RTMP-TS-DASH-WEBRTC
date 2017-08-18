# NGINX MPEG-TS Live Module & Dash JS
## 目录
+   环境搭建   
+   NGINX-RTMP-TS-DASH 直播方案   
+   HTML5 标准
## 环境搭建
+   服务与模块
    +   1、Openresty下载 
    
            https://openresty.org/download/openresty-1.11.2.3.tar.gz
            
    +   2、nginx-ts-module下载 
    
            git clone https://github.com/arut/nginx-ts-module.git
            
    +   3、ffmpeg 下载安装
+   动态编译安装
    +   1、Openresty环境配置
    
            apt-get install libreadline-dev libncurses5-dev libpcre3-dev \
            libssl-dev perl make build-essential
        
    +   2、动态编译安装
        
            ./configure --prefix=/opt/openresty --with-luajit --without-http_redis2_module \
            --with-http_iconv_module --add-dynamic-module=/root/nginx-ts-module
            ...
            make -j4
            ...
            sudo make install
        
    +   3、配置文件
        +   `nginx.conf`
        
                # vim /opt/openresty/nginx/conf/nginx.conf
                error_log  logs/error.log;
                
                pid        logs/nginx.pid;
                
                load_module "/opt/openresty/nginx/modules/ngx_http_ts_module.so"; # 加载模块
    
                events {
                }
                
                http {
                    server {
                        listen 8000;
                
                        location / {
                            root html;
                        }
                
                        location /publish/ {
                            ts;
                            ts_hls path=/var/media/hls segment=10s;
                            ts_dash path=/var/media/dash segment=10s;
                
                            client_max_body_size 0;
                        }
                
                        location /play/ {
                            add_header Cache-Control no-cache;
                            add_header 'Access-Control-Allow-Origin' '*' always;
                            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
                            add_header 'Access-Control-Allow-Headers' 'Range';
                
                            types {
                                application/x-mpegURL m3u8;
                                application/dash+xml mpd;
                                video/MP2T ts;
                                video/mp4 mp4;
                            }
                            alias /var/media/;
                        }
                    }
                }
            
        +   流媒体存放文件夹建立
        
                cd /var & makedir media
                cd media & makedir hls & makedir dash
            
    +   4、FFmpeg推流
    
            ffmpeg -re -i rtmp://live.hkstv.hk.lxdns.com/live/hks -bsf:v h264_mp4toannexb \
            -c copy -f mpegts http://127.0.0.1:8000/publish/sintel
        
    +   5、客户端播放
    
        ```html
        <script src="http://cdn.dashjs.org/latest/dash.all.min.js"></script>
        <style>
            video {
                width: 640px;
                height: 360px;
            }
        </style>
        <div>
            <video data-dashjs-player autoplay src="http://1127.0.0.1:8000/play/dash/sintel/index.mpd" controls></video>
        </div>
        ```
        
    +   6、如果不使用 ffmpeg 直接拉流到`http://127.0.0.1:8000/publish/sintel` 服务的解决方案？ 
        +   （1）nginx-rtmp-module下载 `git clone https://github.com/arut/nginx-rtmp-module.git`
        +   （2）和安装`nginx-ts-module`模块一样动态编译安装既可以，最后别忘记了的在配置文件load `nginx-rtmp-module.so`文件
        +   （3）按照这个顺序：`OBS => nginx-rtmp => nginx-ts`推流，OBS也可以是别的网络推流设备
        +   （4）通过以上我们可以不直接使用ffmpeg 去推流了，而是在Windows端口可以通过OBS很简单的去推流了
        +   （5）待测试...
    +   7、总结，一切顺利通过。    
        
##  NGINX-RTMP-TS-DASH 直播方案
+   编译安装
    +   1、下载nginx-rtmp-module模块：`git clone https://github.com/arut/nginx-rtmp-module.git`
    +   2、配置 --with-http_xslt_module 时提示 the HTTP XSLT module requires the libxml2/libxslt libraries，安装以下：
            
            sudo apt-get install libxml2 libxml2-dev libxslt-dev
            sudo apt-get install libgd2-xpm libgd2-xpm-dev
            
    +   3、通过configure命令生成Makefile文件，为下一步的编译做准备：
    
            ./configure --prefix=/opt/openresty --with-luajit --without-http_redis2_module --with-http_iconv_module \ 
            --with-http_stub_status_module --with-http_xslt_module --add-dynamic-module=/root/nginx-ts-module \
            --add-dynamic-module=/root/nginx-rtmp-module
            
+   `nginx.conf` 配置信息

        # vim /opt/openresty/nginx/conf/nginx.conf
        user  www;
        worker_processes  1;
        
        error_log  logs/error.log;
        
        pid        logs/nginx.pid;
        
        load_module "/opt/openresty/nginx/modules/ngx_http_ts_module.so";
        load_module "/opt/openresty/nginx/modules/ngx_rtmp_module.so";
        
        events {
            worker_connections  1024;
        }
         
         http {
             server {
                 listen 8000;
         
                 # This URL provides RTMP statistics in XML
                 location /stat {
                    rtmp_stat all;
                    rtmp_stat_stylesheet stat.xsl;
                 }
         
                 location /stat.xsl {
                    root html;
                 }
         
                 location /hls {
                     # Serve HLS fragments
                    add_header Cache-Control no-cache;
                    add_header 'Access-Control-Allow-Origin' '*' always;
                    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
                    add_header 'Access-Control-Allow-Headers' 'Range';
                
                    types {
                        application/vnd.apple.mpegurl m3u8;
                        video/mp2t ts;
                    }
                
                    root /tmp;
                 }
        
                 location /dash {
                     # Serve DASH fragments
                     add_header Cache-Control no-cache;
                     add_header 'Access-Control-Allow-Origin' '*' always;
                     add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
                     add_header 'Access-Control-Allow-Headers' 'Range';
         
                     types {
                         application/dash+xml mpd;
                         video/mp4 mp4;
                     }
    
                     root /tmp;
                 }
             }
         }
        
         rtmp {
             listen 1935;
             chunk_size 4000;
             idle_streams off;
             ping 30s;
             notify_method get;
    
             server {
                 listen 1935;
                 chunk_size 4000;
         
                 drop_idle_publisher 10s;
                 idle_streams off;
                
                 application live {
                     live on;
                 }
         
                 application hls {
                     live on;
                     hls on;
                     hls_path /tmp/hls;
                 }
         
                 # MPEG-DASH is similar to HLS
                 application dash {
                     live on;
                     dash on;
                     dash_path /tmp/dash;
                 }
             }
         }
     
+   拷贝xml文件：`cp /root/nginx-rtmp-module/stat.xsl /opt/openresty/nginx/html`    
+   流状态查看：`http://127.0.0.1:8000/stat`    
+   OBS推流地址：`rtmp://127.0.0.1/dash/123`    
+   VLC观看RTMP直播流：`rtmp://127.0.0.1/dash/123`    
+   DASH格式HTTP播放

```html
<script src="http://cdn.dashjs.org/latest/dash.all.min.js"></script>
<style>
    video {
        width: 640px;
        height: 360px;
    }
</style>
<div>
    <video data-dashjs-player autoplay src="http://127.0.0.1:8000/dash/123.mpd" controls></video>
</div>
```
    
+   测试结束    