# own_your_doh_server
搭建属于自己的DNS over HTTPS（DoH）服务器，支持websocket方式反代不影响域名原来的页面服务  
  
  
时下越来越多机友使用DoH服务了，那自己拥有一台专属的DoH服务器是不是很酷呢 ( *￣▽￣)o ─═≡※:☆  

下面分享一个简单的搭建过程，就可以让各位机友美梦成真了，而且这个方法也适合于IPv6 only或IPv4 nat有公网IPv6的小鸡都可以变成DoH服务器，前提是挂CF的CDN，这样IPv4的客户端也可以使用这些DoH服务器，当然你的小鸡是IPv4公网的就更好了，而且这个方法支持websocket方式反代不影响域名原来的页面服务   
   
下面以ubuntu为例子开车  
  
SSH到小鸡，使用root用户并转到根目录  

#cd /   
  
下载dnsproxy开源软件并解压，然后拷贝到缺省的环境目录  
  
#wget https://github.com/AdguardTeam/dnsproxy/releases/download/v0.37.4/dnsproxy-linux-amd64-v0.37.4.tar.gz   
  
#tar -xzvf dnsproxy-linux-amd64-v0.37.4.tar.gz   
  
#mv /linux-amd64/dnsproxy /usr/local/bin  
  
#rm -rf /linux-amd64  
  
这里假设各位机友自己能够安装nginx，certbot，python3-certbot-nginx等包，建议使用apt方式安装，然后自己可以写虚拟主机配置并签证书

如果https的页面已经可以正常访问的话，编辑虚拟主机配置，加入下面的配置内容，就只是location /dns-query的包含部分  
  
------nginx虚拟主机增加的配置，一定要在listen 443行加上http2------
  
location /dns-query {  
   proxy_pass         https://127.0.0.1:10443; # 后续dnsproxy监听的端口号  
   proxy_set_header X-Real-IP $remote_addr; # 传递客户端的真实ip到dnsproxy，否则dnsproxy的edns功能不能用  
   }  
  
配置完成后重启nginx    
   
#systemctl restart nginx  
  
另外假设各位机友自己的小鸡域名为www.yourdomain.com，并在之前签好了证书，那个edns功能是可以使网站在全球分布的服务器根据用户的IP归属回应最近的网站IP地址      
  
------配置dnsproxy的Systemd服务启动方式------  
  
确保是root用户并在根目录,检查是否已经有system目录  
#ls -l /usr/lib/systemd  
  
没有的的话创建system目录  
#mkdir /usr/lib/systemd/system  
  
编辑doh服务文件  
#nano /usr/lib/systemd/system/doh.service  
  
写入以下内容  
  
[Unit]  
Description=doh  
After=network.target  
  
[Service]  
TimeoutStartSec=30  
ExecStart=/usr/local/bin/dnsproxy -l 127.0.0.1 --https-port=10443 --tls-crt=/etc/letsencrypt/live/www.yourdomain.com/fullchain.pem --tls-key=/etc/letsencrypt/live/www.yourdomain.com/privkey.pem -u https://dns.google/dns-query -f 8.8.8.8:53 -f 8.8.4.4:53 --cache --edns -p 0  
ExecStop=/bin/kill $MAINPID  
  
[Install]  
WantedBy=multi-user.target  
  
保存并退出  
  
设置开机启动doh  
#systemctl enable doh  
  
启动doh服务  
#systemctl start doh  
  
检查服务状态       
#systemctl status doh            
          
检查网络连接状态及doh是否有在监听配置文件设定的10443端口      
       
#netstat -tunap    
   
上面dnsproxy的另外一个 -u 参数可以是下面这个替代的，其实各位机友可以找自己喜欢的上游DNS服务器替代进去  
  
ExecStart=/usr/local/bin/dnsproxy -l 127.0.0.1 --https-port=10443 --tls-crt=/etc/letsencrypt/live/www.yourdomain.com/fullchain.pem --tls-key=/etc/letsencrypt/live/www.yourdomain.com/privkey.pem -u https://9.9.9.10/dns-query -f 8.8.8.8:53 -f 8.8.4.4:53 --cache --edns -p 0  
  
现在我们自己拥有的doh服务器地址为:     
  
https://www.yourdomain.com/dns-query  
   
最简单试验是否成功的方法是在firefox，chrome，edge等浏览器里面设置启用基于 HTTPS 的 DNS 自定义，然后把上面我们自己的doh服务器连接连接填进去，完成后访问https://dnsleaktest.com/ 这个网站，进去后点击standard test 或 extended test就可以显示你现在的dns服务器不是本地ISP的   
  
  
  

  
