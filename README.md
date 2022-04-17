### Docker VLESS with XTLS & docker Trojan
Docker 部署 Xray，实现VLESS+XTLS/Trojan共存
## 准备工作
* 一个VPS，如[Vultr](https://www.vultr.com/?ref=6906410) ;[ION](https://ion.krypt.asia/aff.php?aff=139)
* 准备一个域名，并将A记录添加好;

## 1. 安装acme.sh
```
apt-get update && apt-get -y install socat //安装socat

wget -qO- get.acme.sh | bash //安装脚本

source ~/.bashrc //让环境变量生效，以后无论在哪个路径，直接使用acme.sh
```
## 2. 配置证书
```
export CF_Key="XXX" //修改XXX
export CF_Email="XXX@hotmail.com" //修改XXX
acme.sh --register-account -m XXX@hotmail.com --server zerossl //修改XXX
acme.sh --issue --dns dns_cf -d XXX.XXX.com -k ec-256 //修改XXX
mkdir /etc/xray
acme.sh --installcert -d XXX.XXX.com --fullchain-file /etc/xray/xray.crt --key-file /etc/xray/xray.key --ecc //修改XXX
acme.sh --upgrade --auto-upgrade
```
## 3. Docker安装
```
wget -qO- get.docker.com | bash //安装docker
systemctl start docker //启动docker服务
systemctl status docker //查看docker运行状态
docker -v //查看docker版本
systemctl enable docker //将docker服务加入开机自启动
```
## 4. Docker安装xray
```
vi /etc/xray/config.json
```
Xray配置如下：
```
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "Your UUID", // 填写你的 UUID
                        "flow": "xtls-rprx-direct",
                        "level": 0,
                        "email": "XXX@hotmail.com" // 填写你的 EMAIL
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 15005, // 默认回落到 Xray 的 Trojan 协议
                        "xver": 1
                    },
                    {
                        "path": "/ws", // 必须换成自定义的 PATH
                        "dest": 15050,
                        "xver": 1
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/etc/xray/xray.crt", // 换成你的证书，绝对路径
                            "keyFile": "/etc/xray/xray.key" // 换成你的私钥，绝对路径
                        }
                    ]
                }
            }
        },
        {
            "port": 15005,
            "listen": "127.0.0.1",
            "protocol": "trojan",
            "settings": {
                "clients": [
                    {
                        "password": "Your password", // 填写你的密码
                        "level": 0,
                        "email": "XXX@hotmail.com" // 填写你的 EMAIL
                    }
                ],
                "fallbacks": [
                    {
                        "dest": 80 // 或者回落到其它也防探测的代理
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "none",
                "tcpSettings": {
                    "acceptProxyProtocol": true
                }
            }
        },
        {
            "port": 15050,
            "listen": "127.0.0.1",
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "Your UUID", // 填写你的 UUID
                        "level": 0,
                        "email": "XXX@hotmail.com"  // 填写你的 EMAIL
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "ws",
                "security": "none",
                "wsSettings": {
                    "acceptProxyProtocol": true, // 提醒：若你用 Nginx/Caddy 等反代 WS，需要删掉这行
                    "path": "/ws" // 必须换成自定义的 PATH，需要和分流的一致
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```
运行Docker
```
docker run -d --network host --name xray --restart=always -v /etc/xray:/etc/xray teddysun/xray
```


## 5. 配制信息如下：
```
---vless_tcp_xtls 配置信息---
地址 (Address) = test.youdomain.com
端口 (Port) = 443
VLESS ID (UUID) = Your UUID
传输协议 (Network) = tcp  伪装类型 (header type) = none
流控 (Flow) = xtls-rprx-direct
---END---
```
```
---trojan_tcp_tls 配置信息---
地址 (Address) = test.youdomain.com
端口 (Port) = 443
密码 = Your password
传输协议 (Network) = tcp  伪装类型 (header type) = none
加密 (Flow) = tls
---END---
```
```
---vless_ws_tls （支持CDN）配置信息---
地址 (Address) = test.youdomain.com
端口 (Port) = 443
VLESS ID (UUID) = Your UUID
传输协议 (Network) = WebSocket  webSocket路经 (WebSocket Path) = /ws
加密 (Flow) = tls
---END---
```
# 参考与借鉴
https://github.com/XTLS/Xray-examples/tree/main/VLESS-TCP-XTLS-WHATEVER

https://hub.docker.com/r/teddysun/xray
