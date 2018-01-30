# 传出代理

V2Ray 提供了传出代理功能，利用它可以实现中转（在没有中转服务器操作权限的情况下）。

## 基本传出代理

使用传出代理可以实现由一个 Shadowsocks 服务器或者 V2Ray(VMess) 服务器来中转你的网络流量，中转服务器只能看到你加密的数据而不知道原始的数据是什么。

以下面的配置说明，它的工作原理是：

1. 你在 Twitter 发了个帖子 f**k GFW，由 V2Ray 代理
2. V2Ray 客户端收到浏览器发出的 f**k GFW 的帖子后，首先由对其进行加密(VMess，id: b12614c5-5ca4-4eba-a215-c61d642116ce,目的服务器: 1.1.1.1:8888)
3. 加密后数据包将被转到 transit 这个 outbound 中，在这里数据包又会加密一次(Shadowsocks, password: password, 服务器: 2.2.2.2:1024)
4. 两次加密后的数据包被发送到了 Shadowsocks 服务器，该服务器收到后解包后得到仍是加密的数据包（步骤 2 中加密后的数据包），然后将数据包发到 VMess 服务器。即便这个 Shadowsocks 服务器的主人是个偷窥狂魔，他也没办法看到你的原始数据。
5. VMess 服务器收到 Shadowsocks 服务器发来的数据包，解密得到原始的数据包，然后把你这个帖子发到 Twitter 的网站中。

只要第 5 步中的服务器是自己掌控的就不用担心别人看到你的上网的内容。

客户端：

```javascript
{
  "outbound": {
    "protocol": "vmess",
    "settings": { // settings 的根据实际情况修改
      "vnext": [
        {
          "address": "1.1.1.1",
          "port": 8888,
          "users": [
            {
              "alterId": 64,
              "id": "b12614c5-5ca4-4eba-a215-c61d642116ce"
            }
          ]
        }
      ]
    },
    "streamSettings": {
      "network": "tcp"  // 此处不能是 "kcp"，设成 "kcp" 将无法联网
    },
    "proxySettings": {
        "tag": "transit"  // 这里的 tag 必须跟作为代理 VPS 的 tag 一致，这里设定的是 "transit"
      }
  },
  "outboundDetour": [
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "2.2.2.2",
            "method": "aes-256-cfb",
            "ota": false,
            "password": "password",
            "port": 1024
          }
        ]
      },
      "tag": "transit"
    }
  ]
}
```

## 链式传出代理

如果你有多个 Shadowsocks 或 VMess 账户，那么你可以这样:

```javascript
{
  "outbound": {
    "protocol": "vmess",
    "settings": { // settings 的根据实际情况修改
      "vnext": [
        {
          "address": "1.1.1.1",
          "port": 8888,
          "users": [
            {
              "alterId": 64,
              "id": "b12614c5-5ca4-4eba-a215-c61d642116ce"
            }
          ]
        }
      ]
    },
    "streamSettings": {
      "network": "tcp"  
    },
    "tag": "DOUS",
    "proxySettings": {
        "tag": "DOSG"  
      }
  },
  "outboundDetour": [
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "2.2.2.2",
            "method": "aes-256-cfb",
            "ota": false,
            "password": "password",
            "port": 1024
          }
        ]
      },
      "tag": "AliHK"
    },
    {
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "3.3.3.3",
            "method": "aes-256-cfb",
            "ota": false,
            "password": "password",
            "port": 3442
          }
        ]
      },
      "tag": "AliSG",
      "proxySettings": {
          "tag": "AliHK"  
      }
    },
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "4.4.4.4",
            "port": 8462,
            "users": [
              {
                "alterId": 64,
                "id": "b27c24ab-2b5a-433e-902c-33f1168a7902"
              }
            ]
          }
        ]
      },
      "tag": "DOSG",
      "streamSettings": {
        "network": "tcp"  
      },
      "proxySettings": {
          "tag": "AliSG"  
      }
    },
  ]
}
```

那么数据包经过的节点依次为：

PC -> AliHK -> AliSG -> DOSG -> DOUS -> 目标网站

这样的传出代理形成了一条链条，我称之为链式传出代理。

**注意：如果你打算配置(动态)链式传出代理，应当明确几点：**

* `性能`。链式代理使用了多个节点，可能会造成延时、带宽等网络性能问题，并且客户端对每一个加解密的次数取决于代理链的长度，理论上也会有一定的影响。
* `安全`。前文提到，传出代理会一定程度上提高安全性，但安全取决于最弱一环，并不意味着代理链越长就会越安全。如果你需要匿名，请考虑成熟的匿名方案。
