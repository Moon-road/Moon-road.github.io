<h2>安装go环境</h2>
<p><strong>在控制台进入服务器控制界面，依次运行以下命令。</strong></p>
<ul>
<li><strong>安装go</strong></li>
</ul>
<pre><code>apt install -y wget git openssl curl
wget https://golang.google.cn/dl/go1.21.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go &amp;&amp; sudo  tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
</code></pre>
<ul>
<li><strong>将go的路径添加到环境变量</strong></li>
</ul>
<pre><code>export PATH=$PATH:/usr/local/go/bin
</code></pre>
<ul>
<li><strong>检查go是否安装成功</strong></li>
</ul>
<pre><code>go version
</code></pre>
<ul>
<li><strong>将添加环境变量输出到profile文件内，这样每次开机可以自动添加环境变量</strong></li>
</ul>
<pre><code>echo &quot;export PATH=$PATH:/usr/local/go/bin&quot; &gt;&gt; /etc/profile
source /etc/profile
</code></pre>
<ul>
<li><strong>以下命令都要在管理员账户下运行，先进入管理员账户</strong></li>
</ul>
<pre><code>su -
</code></pre>
<ul>
<li><strong>增加go安装的国内镜像，加快go install的安装速度（国内必要）</strong></li>
</ul>
<pre><code>go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
</code></pre>
<h2>安装derper</h2>
<ul>
<li><strong>下载derper</strong></li>
</ul>
<pre><code class="language-bash">go install tailscale.com/cmd/derper@main
</code></pre>
<ul>
<li><strong>进入~/go/pkg/mod/[tailscale]/cmd/derper文件夹内，执行go编译</strong></li>
</ul>
<pre><code class="language-bash">go build -o /etc/derp/derper
</code></pre>
<ul>
<li><strong>编译完成后要修改cert.go文件，注释以下三行代码。</strong>  <strong>cert.go文件位于~/go/pkg/mod/[tailscale]/cmd/derper</strong> <img src="https://raw.githubusercontent.com/Moon-road/Moon-road.github.io/main/images/net-img-1965d90a54b44393a158a448a44c47ce-20240513160550-3sxfcuy.png" alt="在这里插入图片描述" />​</li>
<li><strong>然后再次进入derper文件夹内编译一次</strong></li>
</ul>
<pre><code class="language-bash">go build -o /etc/derp/derper
</code></pre>
<ul>
<li><strong>检查derper是否安装成功，成功的话会有derper文件夹</strong></li>
</ul>
<pre><code class="language-bash">ls /etc/derp
</code></pre>
<h2>配置derper服务器</h2>
<ul>
<li><strong>生成ssl证书，其中CN=derp.test.com中的网址可以任意填写</strong></li>
</ul>
<pre><code class="language-bash">openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout /etc/derp/derp.test.com.key -out /etc/derp/derp.test.com.crt -subj &quot;/CN=derp.test.com&quot; -addext &quot;subjectAltName=DNS:derp.test.com&quot;
</code></pre>
<ul>
<li><strong>生成derper的配置文件</strong></li>
</ul>
<pre><code class="language-bash">sudo nano /etc/systemd/system/derp.service
</code></pre>
<ul>
<li><strong>将以下内容写入到derp.service文件中</strong></li>
</ul>
<pre><code class="language-bash">[Unit]

Description=TS Derper

After=network.target

Wants=network.target

[Service]

User=root

Restart=always

ExecStart=/etc/derp/derper -hostname derp.test.com -a :14662 -http-port 3478 -certmode manual -certdir /etc/derp --verify-clients

RestartPreventExitStatus=1

[Install]

WantedBy=multi-user.target
</code></pre>
<ul>
<li><strong>启动derper</strong></li>
</ul>
<pre><code class="language-bash">systemctl enable derp
systemctl start derp
</code></pre>
<ul>
<li><strong>检验是否设置成功</strong> <strong>在启动derp后可以在浏览器中进入</strong>​<a href="https://ip:PORT/">https://IP:PORT</a>，如果看到以下网页则说明成功。其中IP是第一步中记录的服务器公网IP，PORT是derp.service中设置的，默认为14662<img src="https://raw.githubusercontent.com/Moon-road/Moon-road.github.io/main/images/net-img-3a892bd9e73f4ec9b19356e280123a6e-20240513160550-disnw3z.png" alt="在这里插入图片描述" />​</li>
</ul>
<h2>在服务器上安装taiscale</h2>
<ul>
<li><strong>运行自动安装脚本</strong></li>
</ul>
<pre><code class="language-bash">curl -fsSL https://tailscale.com/install.sh | sh
</code></pre>
<ul>
<li><strong>启动tailscale并登陆</strong></li>
</ul>
<pre><code class="language-bash">tailscale up
</code></pre>
<p><strong>进入登陆网页登陆tailscale账号</strong></p>
<ul>
<li><strong>重启derp服务</strong></li>
</ul>
<pre><code class="language-bash">systemctl daemon-reload
systemctl restart derp
</code></pre>
<h2>在tailscale中增加derper服务器</h2>
<ul>
<li><strong>打开tailscale的网页console，在access control里的’ssh’之前粘贴以下内容：</strong></li>
</ul>
<pre><code class="language-bash">&quot;derpMap&quot;: {
        //不禁用官方的中转节点
        //&quot;OmitDefaultRegions&quot;: true,
        &quot;Regions&quot;: {
            &quot;900&quot;: {
                &quot;RegionID&quot;:   900,
                &quot;RegionCode&quot;: &quot;test&quot;,
                &quot;RegionName&quot;: &quot;Test Derper&quot;,
                &quot;Nodes&quot;: [
                    {
                        &quot;Name&quot;:             &quot;900a&quot;,
                        &quot;RegionID&quot;:         900,
                        &quot;DERPPort&quot;:         12345, //更换为自己的PORT
                        &quot;IPv4&quot;:             &quot;192.168.1.1&quot;, //这里更换为自己的PI
                        &quot;InsecureForTests&quot;: true,
                    },
                ],
            },
            &quot;1&quot;:  null,
            &quot;2&quot;:  null,
            &quot;3&quot;:  null,
            &quot;4&quot;:  null,
            &quot;5&quot;:  null,
            &quot;6&quot;:  null,
            &quot;7&quot;:  null,
            &quot;8&quot;:  null,
            &quot;9&quot;:  null,
            &quot;10&quot;: null,
            &quot;11&quot;: null,
            &quot;12&quot;: null,
            &quot;13&quot;: null,
            &quot;14&quot;: null,
            &quot;15&quot;: null,
            &quot;16&quot;: null,
            &quot;17&quot;: null,
            &quot;18&quot;: null,
            &quot;19&quot;: null,
            //&quot;20&quot;: null,
            &quot;21&quot;: null,
            &quot;22&quot;: null,
            &quot;23&quot;: null,
            &quot;24&quot;: null,
            &quot;25&quot;: null,
        },
    },
</code></pre>
<h2>检测是否配置成功</h2>
<ul>
<li><strong>在自己的电脑上输入以下命令：</strong></li>
</ul>
<pre><code class="language-bash">tailscale netcheck
</code></pre>
<p>**如果在DERP **<a href="https://so.csdn.net/so/search?q=latency&amp;spm=1001.2101.3001.7020">latency</a>中出现自己刚才设置的服务器Test Derper，即为安装成功。</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<p>‍</p>
<pre><code class="language-yaml">// Example/default ACLs for unrestricted connections.
{
	// Declare static groups of users. Use autogroups for all users or users with a specific role.
	// &quot;groups&quot;: {
	//  	&quot;group:example&quot;: [&quot;alice@example.com&quot;, &quot;bob@example.com&quot;],
	// },

	// Define the tags which can be applied to devices and by which users.
	// &quot;tagOwners&quot;: {
	//  	&quot;tag:example&quot;: [&quot;autogroup:admin&quot;],
	// },

	// Define access control lists for users, groups, autogroups, tags,
	// Tailscale IP addresses, and subnet ranges.
	&quot;acls&quot;: [
		// Allow all connections.
		// Comment this section out if you want to define specific restrictions.
		{&quot;action&quot;: &quot;accept&quot;, &quot;src&quot;: [&quot;*&quot;], &quot;dst&quot;: [&quot;*:*&quot;]},
	],
	&quot;derpMap&quot;: {
		&quot;OmitDefaultRegions&quot;: true,
		&quot;Regions&quot;: {
			&quot;910&quot;: {
				&quot;RegionID&quot;:   910,
				&quot;RegionCode&quot;: &quot;jdCloud&quot;,
				&quot;RegionName&quot;: &quot;JD Derper&quot;,
				&quot;Nodes&quot;: [
					{
						&quot;Name&quot;:             &quot;910a&quot;,
						&quot;RegionID&quot;:         910,
						&quot;DERPPort&quot;:         14662,
						&quot;IPv4&quot;:             &quot;116.198.235.10&quot;,
						&quot;InsecureForTests&quot;: true,
					},
				],
			},

			// &quot;913&quot;: {
			// 	&quot;RegionID&quot;:   913,
			// 	&quot;RegionCode&quot;: &quot;raksmart&quot;,
			// 	&quot;RegionName&quot;: &quot;raksmart&quot;,
			// 	&quot;Nodes&quot;: [
			// 		{
			// 			&quot;Name&quot;:             &quot;raksmart&quot;,
			// 			&quot;RegionID&quot;:         913,
			// 			&quot;DERPPort&quot;:         14662, // 自定义的 derper 端口
			// 			&quot;IPv4&quot;:             &quot;142.4.124.3&quot;,
			// 			&quot;InsecureForTests&quot;: true,
			// 		},
			// 	],
			// },
			&quot;914&quot;: {
				&quot;RegionID&quot;:   914,
				&quot;RegionCode&quot;: &quot;bytevirt&quot;,
				&quot;RegionName&quot;: &quot;bytevirt&quot;,
				&quot;Nodes&quot;: [
					{
						&quot;Name&quot;:             &quot;bytevirt&quot;,
						&quot;RegionID&quot;:         914,
						&quot;DERPPort&quot;:         36602, // 自定义的 derper 端口
						&quot;HostName&quot;:         &quot;natsg1.bytevirt.net&quot;,
						&quot;InsecureForTests&quot;: true,
					},
				],
			},
		},
	},
	// Define users and devices that can use Tailscale SSH.
	&quot;ssh&quot;: [
		// Allow all users to SSH into their own devices in check mode.
		// Comment this section out if you want to define specific restrictions.
		{
			&quot;action&quot;: &quot;check&quot;,
			&quot;src&quot;:    [&quot;autogroup:member&quot;],
			&quot;dst&quot;:    [&quot;autogroup:self&quot;],
			&quot;users&quot;:  [&quot;autogroup:nonroot&quot;, &quot;root&quot;],
		},
	],

	// Test access rules every time they're saved.
	// &quot;tests&quot;: [
	//  	{
	//  		&quot;src&quot;: &quot;alice@example.com&quot;,
	//  		&quot;accept&quot;: [&quot;tag:example&quot;],
	//  		&quot;deny&quot;: [&quot;100.101.102.103:443&quot;],
	//  	},
	// ],
}

</code></pre>
<p>‍</p>
