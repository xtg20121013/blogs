现在越来越多主流站点都已经全站Https化了，国外有些国家地区甚至立法强制使用https，足以说明https的重要性了。但是现在ssl证书太贵了，甚至比服务器还贵，小型网站或者个人网站使用成本太高。[Let's Encrypt](https://letsencrypt.org/)这个组织提供了免费的证书以及获取证书的工具[certbot](https://certbot.eff.org/)，其实官网已经介绍的很清楚，我这就当个简单翻译吧。
####一、为什么用HTTPS
HTTPS缺点：

1. 通信要加密，服务器和客户端加密解密要占用额外资源。
2. 全站所有资源都必须是https的，否则浏览器会报警，尤其注意外链图片也要是https的。
3. 如果是老系统切换，肯定有很多相关联的系统要做测试适配，工程量巨大。
4. 价格贵，贵，贵（重要的事说三遍），小网站用不起啊。

Https优势：

1. 防止运营商插入广告，有没有想起用流量访问页面时经常弹出的绿色流量监控的小球，就是被移动拦截注入的。
2. 更严重的，被嵌入的代码有问题，导致本来正常的页面报错，甚至带来病毒。
3. 用户数据泄露。
4. 出于安全考虑，苹果已经要求所有app的外链请求都必须是https的，微信小程序也是，这肯定也是未来趋势了。

大公司不差钱的，最好还是买一个，收费证书一般带保险，客户可以查看到具体的证书信息，越贵的证书，对应的机构安全性越高，认证的企业也越可靠，总有客户在意这些。

差钱的可以用我这次推荐的这个[Let's Encrypt](https://letsencrypt.org/)的免费证书。

####二、用Let's Encrypt免费证书
以linux+nginx为例，尽量介绍通用点的方法。

- 下载certbot脚本

		wget https://dl.eff.org/certbot-auto	
		chmod a+x certbot-aut
		
- 校验域名，获取证书

		$ sudo ./path/to/certbot-auto certonly --webroot -w /var/www/example -d example.com -d www.example.com -w /var/www/thing -d thing.is -d m.thing.is
	一个-w 站点根目录 对应一个-d 域名，为了保证这个域名归你所属，certbot会做一次校验，比如随机在你的站点根目录下的/.well-known/acme-challenge目录里放一个xxxx文件，然后通过访问http://域名/.well-known/acme-challenge/xxxx 检验文本内容和请求到的内容是否一致。
	
	如果证书获取成功就会在/etc/letsencrypt/live/你的域名 目录下。
	
- 还需要在服务器生成dhparam
	
	    openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

- 配置nginx.conf

	在自己的server标签内加上
	
		listen       443 ssl; # https默认443端口
        server_name  blog.52xtg.com; # 填自己的域名，后面都把该域名替换为自己的
	    ssl_certificate /etc/letsencrypt/live/blog.52xtg.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/blog.52xtg.com/privkey.pem;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers  on;
		
	一般还要把http的默认转发到https上，另开个server节点，监听80端口
	
		server {
        	listen       80;
        	server_name  blog.52xtg.com; # 填自己的域名，后面都把该域名替换为自己的

			location / {
	    		return 301 https://blog.52xtg.com$request_uri; # 用301永久跳转
			}
    	}
    	
- 重启nginx应该就能访问到了
		
####三、自动更新证书

certbot还有更新证书的命令。注意，该命令只有在证书过期后才会更新，否则都会被官方忽略的并不会真正更新证书有效期。

	./path/to/certbot-auto renew --no-self-upgrade
		
将该命令注册到crontab定时任务中，官方建议一天二次。


最后，祝大家都能愉快的用上https。

欢迎来我的个人博客逛逛: [https://blog.52xtg.com](https://blog.52xtg.com)