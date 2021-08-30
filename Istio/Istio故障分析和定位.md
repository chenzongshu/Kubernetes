Istio 提供了两个非常有价值的命令来帮助诊断流量管理配置相关的问题，[`proxy-status`](https://istio.io/latest/zh/docs/reference/commands/istioctl/#istioctl-proxy-status) 和 [`proxy-config`](https://istio.io/latest/zh/docs/reference/commands/istioctl/#istioctl-proxy-config) 命令。`proxy-status` 命令容许您获取网格的概况，并识别出导致问题的代理。`proxy-config` 可以被用于检查 Envoy 配置和诊断问题



# 获取网格概况

`proxy-status` 命令获取网格的概况。如果怀疑某一个 sidecar 没有接收到配置或配置不同步时可用

```bash
istioctl proxy-status
NAME                                                   CDS        LDS        EDS        RDS          ISTIOD                      VERSION
details-v1-558b8b4b76-qzqsg.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
istio-ingressgateway-66c994c45c-cmb7x.istio-system     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-6cf8d4f9cb-wm7x6     1.7.0
productpage-v1-6987489c74-nc7tj.default                SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
prometheus-7bdc59c94d-hcp59.istio-system               SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
ratings-v1-7dc98c7588-5m6xj.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
reviews-v1-7f99cc4496-rtsqn.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
reviews-v2-7d79d5bd5d-tj6kf.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
reviews-v3-7dbcdcbc56-t8wrx.default                    SYNCED     SYNCED     SYNCED     SYNCED       istiod-6cf8d4f9cb-wm7x6     1.7.0
```

如果列表中缺少代理，这意味着它目前没有连接到 Istiod 实例，因此不会接收任何配置。

- `SYNCED` 意思是 Envoy 知晓了 Istiod 已经将最新的配置发送给了它。
- `NOT SENT` 意思是 Istiod 没有发送任何信息给 Envoy。这通常是因为 Istiod 没什么可发送的。
- `STALE` 意思是 Istiod 已经发送了一个更新到 Envoy，但还没有收到应答。这通常意味着 Envoy 和 Istiod 之间存在网络问题，或者 Istio 自身的 bug。

# 检查Envoy和Istiod配置

通过Pod Name，`proxy-status` 命令还可以用来检查 Envoy 已加载的配置和 Istiod 发送给它的配置有什么异同

```
# istioctl -n istio-test proxy-status details-v1-66b6955995-rnjnl
Clusters Match
Listeners Match
Routes Match (RDS last loaded at Thu, 26 Aug 2021 20:13:56 CST)
```

上面就表示是匹配的，如果有不匹配会显示

# 深入Envoy配置

`proxy-config` 命令可以用来查看给定的 Envoy 是如何配置的

下面来看个例子来理解 Envoy 集群、监听器、路由、endpoints 以及它们是如何交互的

如果在一个 Pod 上查询监听器概要信息，可以看到到 Istio 生成了下面的监听器：

- `0.0.0.0:15001` 监听器接收所有进出 Pod 的流量，然后转发请求给一个虚拟监听器。
- 每个服务 IP 一个虚拟监听器，针对每一个非 HTTP 的外部 TCP/HTTPS 流量。
- Pod IP 上的虚拟监听器，针对内部流量暴露的端口。
- `0.0.0.0` 监听器，针对外部 HTTP 流量的每个 HTTP 端口。

```bash
# istioctl proxy-config listeners -n istio-test productpage-v1-5d9b4c9849-99kff
ADDRESS         PORT  MATCH                                                                                           DESTINATION
192.168.242.49  22    Trans: raw_buffer; App: HTTP                                                                    Route: websocket.demo-websocket.svc.cluster.local:22
192.168.242.49  22    ALL                                                                                             Cluster: outbound|22||websocket.demo-websocket.svc.cluster.local
192.168.0.10    53    ALL                                                                                             Cluster: outbound|53||kube-dns.kube-system.svc.cluster.local
192.168.233.126 53    ALL                                                                                             Cluster: outbound|53||consul-consul-dns.default.svc.cluster.local
0.0.0.0         80    Trans: raw_buffer; App: HTTP                                                                    Route: 80
0.0.0.0         80    ALL                                                                                             PassthroughCluster
192.168.100.130 80    Trans: raw_buffer; App: HTTP                                                                    Route: kustomize-guestbook-ui.test.svc.cluster.local:80
192.168.100.130 80    ALL                                                                                             Cluster: outbound|80||kustomize-guestbook-ui.test.svc.cluster.local
······
0.0.0.0         15001 ALL                                                                                             PassthroughCluster
0.0.0.0         15001 Addr: *:15001                                                                                   Non-HTTP/Non-TCP

```

每一个 sidecar 有一个绑定到 `0.0.0.0:15001` 的监听器，来确定 IP 表将所有进出 Pod 的流量路由到哪里。监听器设置 `useOriginalDst` 为 true 意味着它将请求传递给最适合原始请求目的地的监听器。如果找不到匹配的虚拟监听器，它会将请求发送到直接连接到目的地的 `PassthroughCluster`。

如果我们的请求是到端口 `9080` 的出站 HTTP 请求，它将被传递给 `0.0.0.0:9080` 的虚拟监听器。这一监听器将检索在它配置的 RDS 里的路由配置。在这个例子中它将寻找 Istiod（通过 ADS）配置在 RDS 中的路由 `9080`。

```bash
# istioctl -n istio-test proxy-config listeners productpage-v1-5d9b4c9849-99kff -o json --address 0.0.0.0 --port 9080
                            "rds": {
                                "configSource": {
                                    "ads": {},
                                    "initialFetchTimeout": "0s",
                                    "resourceApiVersion": "V3"
                                },
                                "routeConfigName": "9080"
                            },
```

对每个服务，`9080` 路由配置只有一个虚拟主机。我们的请求会走到 reviews 服务，因此 Envoy 将选择一个虚拟主机把请求匹配到一个域。一旦匹配到，Envoy 会寻找请求匹配到的第一个路由。本例中我们没有设置任何高级路由规则，因此路由会匹配任何请求。这一路由告诉 Envoy 发送请求到 `outbound|9080||reviews.default.svc.cluster.local` 集群。

```bash
# istioctl -n istio-test proxy-config routes productpage-v1-5d9b4c9849-99kff --name 9080 -o json
[
    {
        "name": "9080",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "name": "allow_any",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s",
                            "maxGrpcTimeout": "0s"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
```

再去看对应的cluster的

# 查看Envoy版本

```bash
kubectl -n istio-test exec -it  details-v1-66b6955995-rnjnl -c istio-proxy -- pilot-agent request GET server_info --log_as_json | jq {version}
{
   "version": "494a674e70543a319ad4865482c125581f5746bf/1.19.0/Clean/RELEASE/BoringSSL"
}
```

