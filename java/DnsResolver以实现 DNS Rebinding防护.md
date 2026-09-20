
DNS Rebinding 攻击的核心问题是:应用在**校验阶段**解析域名得到一个公网 IP,通过校验后,攻击者控制的 DNS 服务器在**连接阶段**把同一域名重新解析到内网/回环地址(TTL 极短甚至为 0),导致请求实际打到内部服务上。

防护的关键是**解析与连接必须使用同一个 IP(避免 TOCTOU),并且对解析出的每一个 IP 都做地址校验(拒绝私有段/回环/链路本地等)**,而不是只校验域名本身。

下面是一个 Java 实现思路,基于自定义 `DnsResolver`(比如给 Apache HttpClient 用)加白名单/黑名单校验:

```java
import org.apache.hc.client5.http.DnsResolver;
import org.apache.hc.client5.http.SystemDefaultDnsResolver;

import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Arrays;
import java.util.function.Predicate;

public class SafeDnsResolver implements DnsResolver {

    private final DnsResolver delegate = new SystemDefaultDnsResolver();

    // 可按需要收紧/放宽,例如内部服务需要访问某些私网段时加白名单例外
    private static final Predicate<InetAddress> IS_FORBIDDEN = addr ->
            addr.isLoopbackAddress()        // 127.0.0.1, ::1
            || addr.isLinkLocalAddress()    // 169.254.x.x, fe80::
            || addr.isSiteLocalAddress()    // 10.x, 172.16-31.x, 192.168.x
            || addr.isAnyLocalAddress()     // 0.0.0.0
            || addr.isMulticastAddress();

    @Override
    public InetAddress[] resolve(String host) throws UnknownHostException {
        InetAddress[] resolved = delegate.resolve(host);

        if (resolved.length == 0) {
            throw new UnknownHostException("No addresses resolved for host: " + host);
        }

        // 关键点1:对解析出的每个地址都做校验,而不是只查第一个
        for (InetAddress addr : resolved) {
            if (IS_FORBIDDEN.test(addr)) {
                throw new UnknownHostException(
                    "Blocked potentially unsafe address for host " + host + ": " + addr.getHostAddress()
                    + " (possible DNS rebinding)");
            }
        }

        // 关键点2:这里返回的 IP 数组会被 HttpClient 直接用于建立连接,
        // 而不是连接时再重新解析一次域名,从而避免"校验用一个IP、连接用另一个IP"的 TOCTOU 窗口
        return resolved;
    }

    @Override
    public String resolveCanonicalHostname(String host) throws UnknownHostException {
        return delegate.resolveCanonicalHostname(host);
    }
}
```

配合 HttpClient 使用:

```java
PoolingHttpClientConnectionManager cm =
    PoolingHttpClientConnectionManagerBuilder.create()
        .setDnsResolver(new SafeDnsResolver())
        .build();

CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

几个要点补充说明:

1. **为什么要拦截在 `resolve()` 里**:大部分 HTTP 客户端库(HttpClient5、OkHttp 的 `Dns` 接口、Netty 的 `AddressResolverGroup` 等)在建连时会调用这个接口拿到 IP 再直接 connect,不会二次解析域名——所以只要这里返回的 IP 是校验过的,后续连接就一定用的是这个 IP,天然避免了 rebinding 的"二次解析"窗口。

2. **IPv6 同样要防**:`isSiteLocalAddress()`、`isLoopbackAddress()` 等 API 对 IPv4/IPv6 都适用,但要注意 IPv4-mapped IPv6 地址(`::ffff:127.0.0.1`)有时会绕过朴素判断,建议加一层显式检查或直接禁用 IPv4-mapped 形式。

3. **如果是内部系统需要访问内网服务**:不要整体放开黑名单,而是维护一个基于业务需要的**目标域名/IP 白名单**,校验逻辑改成"不在白名单内的私网地址一律拒绝",比单纯放开更安全。

4. **OkHttp 的等价实现**是实现 `okhttp3.Dns` 接口,逻辑一样;Netty 则是自定义 `AddressResolverGroup`/`DnsAddressResolverGroup` 并在 resolve 回调里过滤。

5. **别忘了连接超时后的重试路径**:如果客户端在连接失败后会自动重新解析重试,要确认重试逻辑走的还是这个自定义 resolver,而不是绕过了它直接调用 `InetAddress.getByName`。

