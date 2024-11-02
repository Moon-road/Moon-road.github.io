<h2>1、Fork Tailscale 代码到自己仓库</h2>
<p><a href="https://link.csdn.net/?target=https%3A%2F%2Fgithub.com%2Ftailscale%2Ftailscale%3Flogin%3Dfrom_csdn">tailscale代码仓库</a></p>
<h2>2、 找到 tailscale 代码中的 cmd/derper/cert.go 文件，将与域名验证相关的内容删除或注释：</h2>
<p>修改之后记得提交代码</p>
<pre><code>func (m *manualCertManager) getCertificate(hi *tls.ClientHelloInfo) (*tls.Certificate, error) {
	//if hi.ServerName != m.hostname {
	//	return nil, fmt.Errorf(&quot;cert mismatch with hostname: %q&quot;, hi.ServerName)
	//}
	return m.cert, nil
}
</code></pre>
<h2>3、编写创建自签名证书，通过以下build_cert.sh脚本：</h2>
<pre><code># build_cert.sh

#!/bin/bash

CERT_HOST=$1
CERT_DIR=$2
CONF_FILE=$3

echo &quot;[req]
default_bits  = 2048
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = XX
stateOrProvinceName = N/A
localityName = N/A
organizationName = Self-signed certificate
commonName = $CERT_HOST: Self-signed certificate

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = $CERT_HOST
&quot; &gt; &quot;$CONF_FILE&quot;

mkdir -p &quot;$CERT_DIR&quot;
openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout &quot;$CERT_DIR/$CERT_HOST.key&quot; -out &quot;$CERT_DIR/$CERT_HOST.crt&quot; -config &quot;$CONF_FILE&quot;

</code></pre>
<h2>4、编写 Dockerfile，将 MODIFIED_DERPER_GIT 的设置为自己的tailscale代码仓库git地址，BRANCH设置为代码分支</h2>
<pre><code>FROM golang:latest AS builder

WORKDIR /app

# ========= CONFIG =========
# - download links
ENV MODIFIED_DERPER_GIT=https://github.com/xxxxx/tailscale.git
ENV BRANCH=main
# ==========================

# build modified derper
RUN git clone -b $BRANCH $MODIFIED_DERPER_GIT tailscale --depth 1 &amp;&amp; \
    cd /app/tailscale/cmd/derper &amp;&amp; \
    /usr/local/go/bin/go build -ldflags &quot;-s -w&quot; -o /app/derper &amp;&amp; \
    cd /app &amp;&amp; \
    rm -rf /app/tailscale

FROM ubuntu:20.04
WORKDIR /app

# ========= CONFIG =========
# - derper args
ENV DERP_HOST=127.0.0.1
ENV DERP_CERTS=/app/certs/
ENV DERP_STUN true
ENV DERP_VERIFY_CLIENTS false
# ==========================

# apt
RUN apt-get update &amp;&amp; \
    apt-get install -y openssl curl

COPY build_cert.sh /app/
COPY --from=builder /app/derper /app/derper

# build self-signed certs &amp;&amp; start derper
CMD bash /app/build_cert.sh $DERP_HOST $DERP_CERTS /app/san.conf &amp;&amp; \
    /app/derper --hostname=$DERP_HOST \
    --certmode=manual \
    --certdir=$DERP_CERTS \
    --stun=$DERP_STUN  \
    --verify-clients=$DERP_VERIFY_CLIENTS

</code></pre>
<h2>5、构建docker镜像</h2>
<p>构建镜像可能需要科学上网，下载一些依赖包。<br />
！！！注意build_cert.sh和Dockerfile应该在在同一目录下<br />
进入Dockerfile所在目录下执行构建镜像命令</p>
<pre><code>docker build -t derper:V1.0 . 
</code></pre>
<h2>6、运行docker derper 容器，并将端口映射出来</h2>
<p>将443和3478端口映射到宿主机上；</p>
<pre><code>docker run --restart always --name derper -p 4435:443 -p 3478:3478/udp  -d &lt;derper镜像id&gt;
</code></pre>
<p>云服务器若是开启了防火墙记得开放相关端口，云服的安全策略组也要开放端口。</p>
<h2>7、查看容器日志</h2>
<pre><code>docker logs -f derper
Generating a RSA private key
.......................................+++++
..............+++++
writing new private key to '/app/certs//127.0.0.1.key'
-----
2023/02/26 14:30:31 no config path specified; using /var/lib/derper/derper.key
2023/02/26 14:30:31 derper: serving on :443 with TLS
2023/02/26 14:30:31 running STUN server on [::]:3478
</code></pre>
<p>tailscale netcheck 实际上只检测 3478/udp 的端口， 就算 netcheck 显示能连，也不一定代表 4435端口可以转发流量。最简单的办法是直接打开 DERP 服务器的 URL：https://ip:4435，如果看到如下页面，且地址栏的 SSL 证书标签显示正常可用，那才是真没问题了。<br />
​<img src="https://raw.githubusercontent.com/Moon-road/Moon-road.github.io/main/images/network-asset-8ecc1351830f1badbf5c78015ad3f9f9-20241102104850-1bp8o1n.png" alt="在这里插入图片描述" />​</p>
<h2>8、Headscale 的配置中需要将 DERP</h2>
<p>我们在 Headscale 的配置中需要将 DERP 的域名设置为 IP！除了 derper 之外，Tailscale 客户端还需要跳过域名验证，这个需要在 DERP 的配置中设置。而 Headscale 的本地 YAML 文件目前还不支持这个配置项，所以没办法，咱只能使用在线 URL 了。JSON 配置内容如下：</p>
<pre><code>{
  &quot;Regions&quot;: {
    &quot;901&quot;: {
      &quot;RegionID&quot;: 901,
      &quot;RegionCode&quot;: &quot;ali-sh&quot;,
      &quot;RegionName&quot;: &quot;Aliyun Shanghai&quot;,
      &quot;Nodes&quot;: [
        {
          &quot;Name&quot;: &quot;901a&quot;,
          &quot;RegionID&quot;: 901,
          &quot;DERPPort&quot;: 4435,
          &quot;HostName&quot;: &quot;xxxx&quot;,
          &quot;IPv4&quot;: &quot;xxxx&quot;,
          &quot;InsecureForTests&quot;: true
        }
      ]
    }
  }
}

</code></pre>
<p>配置解析：</p>
<pre><code>1、HostName 直接填 derper 的公网 IP，即和 IPv4 的值相同。
2、InsecureForTests 一定要设置为 true，以跳过域名验证。
3、Regions 是对象，下面的每一个对象表示一个可用区，每个可用区里面可设置多个 DERP 节点，即 Nodes。
4、每个可用区的 RegionID 不能重复。
5、每个 Node 的 Name 不能重复。
6、RegionName 一般用来描述可用区，RegionCode 一般设置成可用区的缩写
7、DERPPort为docker运行的derper服务443端口映射出来的端口
上面的配置中域名和 IP 部分，你需要根据你的实际情况填写
</code></pre>
<p>你需要把这个 JSON 文件变成 Headscale 服务器可以访问的 URL，比如在 Headscale 主机上搭个 Nginx（<a href="https://link.csdn.net/?target=https%3A%2F%2Fblog.csdn.net%2Fgod_sword_%2Farticle%2Fdetails%2F129341469%3Flogin%3Dfrom_csdn">nginx配置</a>），或者上传到对象存储（比如阿里云 OSS）。</p>
<h2>9、 修改 Headscale 的配置文件</h2>
<p>编辑config.yaml</p>
<pre><code>vim /etc/headscale/config.yaml
</code></pre>
<p>注意urls配置：</p>
<pre><code># /etc/headscale/config.yaml
derp:
  # List of externally available DERP maps encoded in JSON
  urls:
  #  - https://controlplane.tailscale.com/derpmap/default
  #http还是https根据自己的nginx或者是oss来配置
    - https://xxxxx/derp.json

  # Locally available DERP map files encoded in YAML
  #
  # This option is mostly interesting for people hosting
  # their own DERP servers:
  # https://tailscale.com/kb/1118/custom-derp-servers/
  #
  # paths:
  #   - /etc/headscale/derp-example.yaml
  paths: []
   # - /etc/headscale/derp.yaml

  # If enabled, a worker will be set up to periodically
  # refresh the given sources and update the derpmap
  # will be set up.
  auto_update_enabled: true

  # How often should we check for DERP updates?
  update_frequency: 24h

</code></pre>
<p>修改完配置后，重启 headscale 服务：</p>
<pre><code>systemctl restart headscale
</code></pre>
<p>Tailscale 客户端上使用以下命令查看目前可以使用的 DERP 服务器</p>
<pre><code>tailscale netcheck

Report:
	* UDP: true
	* IPv4: yes, 192.168.100.1:49656
	* IPv6: no
	* MappingVariesByDestIP: true
	* HairPinning: false
	* PortMapping: UPnP
	* Nearest DERP: Home Hangzhou
	* DERP latency:
		- home: 9.7ms   (Home Hangzhou)
		-  hs: 25.2ms  (Huawei Shanghai)
		- thk: 43.5ms  (Tencent Hongkong)

</code></pre>
<p>再次查看与通信对端的连接方式：</p>
<pre><code>tailscale status
                coredns              default      linux   active; direct xxxx:45986; offline, tx 131012 rx 196020
                oneplus-8t           default      android active; relay &quot;home&quot;; offline, tx 211900 rx 22780
                openwrt              default      linux   active; direct 192.168.100.254:41641; offline, tx 189868 rx 4074772

</code></pre>
<p>可以看到这一次 Tailscale 自动选择了一个线路最优的国内的 DERP 服务器作为中继，可以测试一下延迟：</p>
<pre><code>tailscale ping 10.1.0.8
pong from oneplus-8t (10.1.0.8) via DERP(home) in 30ms
pong from oneplus-8t (10.1.0.8) via DERP(home) in 45ms
pong from oneplus-8t (10.1.0.8) via DERP(home) in 30ms
</code></pre>
<h2>10、防止 DERP 被白嫖</h2>
<p>默认情况下 DERP 服务器是可以被白嫖的，只要别人知道了你的 DERP 服务器的地址和端口，就可以为他所用。如果你的服务器是个小水管，用的人多了可能会把你撑爆，因此我们需要修改配置来防止被白嫖。</p>
<p>特别声明：只有使用域名的方式才可以通过认证防止被白嫖，使用纯 IP 的方式无法防白嫖，你只能小心翼翼地隐藏好你的 IP 和端口，不能让别人知道。</p>
<p>只需要做两件事情：</p>
<p>1、在 DERP 服务器上安装 Tailscale。</p>
<p>第一步需要在 DERP 服务所在的主机上安装 Tailscale 客户端，启动 tailscaled 进程。</p>
<p>2、derper 启动时加上参数 --verify-clients。<br />
默认情况下 --verify-clients 参数设置的是 false。我们不需要对 Dockerfile 内容做任何改动，只需在容器启动时加上环境变量即可，将之前的启动命令修改一下：</p>
<pre><code>docker run --restart always --name derper -p 4435:443 -p 3478:3478/udp   -e DERP_VERIFY_CLIENTS=true  -d  &lt;derper镜像id&gt;
</code></pre>
<p>！！！自己的服务器IP和端口还是尽量隐藏好</p>
