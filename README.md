# Easier-mosdns

## 自用版本打包（4.18,Windows x64）
[Download Link](https://github.com/c2xvi/easier-mosdns/raw/main/archives/mosdns.7z)

[FastGit Mirror](https://hub.fastgit.xyz/c2xvi/easier-mosdns/raw/main/archives/mosdns.7z)

## 自用心得（Windows）

>注意:需要用到Windows PowerShell<br>
>***如果按照我的方法进行配置却出现了问题，请善用日志文件和`.\mosdns -h`自查一遍问题再提出issue，谢谢***

### 直接使用
- 1、打开PowerShell（管理员）,cd到放置mosdns文件的目录（下文以`D:\mosdns`指代）
```
 cd D:\mosdns
```
- 2、键入如下命令，回车
```
 .\mosdns start -c D:\mosdns\config_mosdns.yaml -d D:\mosdns
```
- 3、如出现
    `2022-01-01T18:00:00.00+0800    info    coremain/run.go:107     working directory changed       {"path": "D:\mosdns"}`
    且没有其他信息即代表成功启动
### 安装为系统服务
- 1、打开PowerShell（管理员）,cd到`D:\mosdns`
- 2、键入如下命令，回车
```
.\mosdns service install -d D:\mosdns -c D:\mosdns\config_mosdns.yaml
再输入：
.\mosdns service start
```
- 3、如果出现以下信息即代表启动成功
```
2022-01-01T18:00:00.000+0800    info    coremain/service.go:130 service is starting
2022-01-01T18:00:00.000+0800    info    coremain/service.go:138 service is running
```
<br>

***如果按照我的方法进行配置却出现了问题，请善用日志文件和`.\mosdns -h`自查一遍问题再提出issue，谢谢***

## 半自动脚本

> 咕咕中

## 自动分流+广告屏蔽配置（借助v2ray-rules-dat中的`geosite.dat`和`geoip.dat`）
```
log:
  level: debug
  file: "mosdns.log"

data_providers:
  - tag: geosite
    file: ./geosite.dat
    auto_reload: true
  - tag: geoip
    file: ./geoip.dat
    auto_reload: true

plugins:
  # 缓存
  - tag: cache
    type: cache
    args:
      size: 5120

  # 转发至本地服务器的插件
  - tag: forward_local
    type: fast_forward
    args:
      upstream:
        - addr: 'https://223.5.5.5/dns-query'
          trusted: true
          dial_addr: '223.5.5.5:443'

  # 转发至远程服务器的插件
  - tag: forward_remote
    type: fast_forward
    args:
      upstream:
        - addr: 'https://dns.google/dns-query'
          trusted: true
          dial_addr: '8.8.8.8:443'

  # 匹配本地域名的插件
  - tag: query_is_local_domain
    type: query_matcher
    args:
      domain:
        - 'provider:geosite:cn'

  # 匹配非本地域名的插件
  - tag: query_is_non_local_domain
    type: query_matcher
    args:
      domain:
        - 'provider:geosite:geolocation-!cn'

  # 匹配广告域名的插件
  - tag: query_is_ad_domain
    type: query_matcher
    args:
      domain:
        - 'provider:geosite:category-ads-all'

  # 匹配本地 IP 的插件
  - tag: response_has_local_ip
    type: response_matcher
    args:
      ip:
        - 'provider:geoip:cn'

  # 主要的运行逻辑插件
  # sequence 插件中调用的插件 tag 必须在 sequence 前定义，
  # 否则 sequence 找不到对应插件。
  - tag: main_sequence
    type: sequence
    args:
      exec:
        # 缓存
        - cache

        # 屏蔽广告域名
        - if: query_is_ad_domain
          exec:
            - _new_nxdomain_response
            - _return

        # 已知的本地域名用本地服务器解析
        - if: query_is_local_domain
          exec:
            - forward_local
            - _return

        # 已知的非本地域名用远程服务器解析
        - if: query_is_non_local_domain
          exec:
            - forward_remote
            - _return

          # 剩下的未知域名用 IP 分流。
          # 这里借助了 `fallback` 工作机制。分流原理请参考 `fallback`
          # 的工作流程。
          # primary 从本地服务器获取应答，丢弃非本地 IP 的结果。
        - primary:
            - forward_local
            - if: "(! response_has_local_ip) && [_response_valid_answer]"
            # - if: "[_response_valid_answer]"
              exec:
                - _drop_response
          # secondary 从远程服务器获取应答。
          secondary:
            - forward_remote
          # 这里建议设置成 local 服务器正常延时的 2~5 倍。
          # 这个延时保证了 local 延时偶尔变高时，其结果不会被 remote 抢答。
          # 如果 local 超过这个延时还没响应，可以假设 local 出现了问题。
          # 这时用就采用 remote 的应答。单位: 毫秒。
          fast_fallback: 200

servers:
  - exec: main_sequence
    listeners:
      - protocol: udp
        addr: 127.0.0.1:1053
      - protocol: tcp
        addr: 127.0.0.1:1053
# API 入口设置     
api:
  http: "127.0.0.1:8080"
```

## 鸣谢
- [ItiineSistiana/mosdns](https://github.com/IrineSistiana/mosdns) : GNU General Public License v3.0
