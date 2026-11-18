# 从零折腾记录：将免费域名托管至 HE.net DNS 并绑定 GitHub Pages

最近领取了一个免费二级域名 `nodeai.loc.cc`（Yunamo 面板），由于没有现成的独立服务器 IP，且面板提示“长期健康检查失败将在 28 天内自动删除”，于是决定把域名的 DNS 托管到老牌免费的 **Hurricane Electric Free DNS（he.net）**，再免费绑定到 **GitHub Pages**，既能建站又能通过健康检查。

整个过程虽然不复杂，但中途踩了不少 **DNS 拼接** 和 **负缓存（Negative Cache）** 的坑。特此记录下来，供以后参考。

---

## 整体思路

1. 在原域名平台配置验证（NS 委派或 TXT 验证），让 **HE.net** 接管域名。
2. 在 **HE.net** 添加 GitHub Pages 的官方 IP（`A` 记录）和 `www` 别名（`CNAME` 记录）。
3. 在 **GitHub 仓库** 开启 Pages，绑定自定义域名并开启 HTTPS。
4. 在原域名平台将名称服务器切回 HE.net，等待全球生效。

---

## 第一步：将域名添加到 HE.net（含避坑）

登录 [dns.he.net](https://dns.he.net/)，点击左侧 **Add a new domain** 输入 `nodeai.loc.cc`。

如果此时域名还在原服务商（`ns1.nodelook.com` / `ns2.nodelook.com`）下，HE.net 会报错提示：
> `ERROR: Delegation was not found. Please delegate to ns1, ns2, ns3, ns4 and ns5.he.net then retry.`

同时下方会给出一条备用的 **TXT 验证记录**：
- **Name**：`dnshenet-key.nodeai.loc.cc`
- **Value**：`5p7OvYn2MbPhPc+RFyDEsKMKS5AruBfgiRd9AQ9dEjw`（以页面实际生成为准）

### 踩坑记录 1：面板自动拼接后缀导致名称重复
在 Yunamo 的 DNS 记录面板添加 TXT 记录时，“名称”栏会**自动拼接** `.nodeai.loc.cc` 后缀。
- **错误填法**：填 `dnshenet-key.nodeai.loc.cc` → 实际生效为 `dnshenet-key.nodeai.loc.cc.nodeai.loc.cc`
- **正确填法**：只填 **`dnshenet-key`**

### 踩坑记录 2：注意字符大小写与字体混淆
部分面板字体中大写字母 `O` 和数字 `0` 非常相似，**务必直接复制粘贴**验证值，不要手动输入。

### 踩坑记录 3：DNS 负缓存（Negative Cache）
如果在记录还没同步或名称填错时就去点 HE.net 验证，HE.net 的递归解析器会把“该记录不存在（NXDOMAIN）”的结果缓存下来。
根据原域名的 SOA 参数，这个负缓存最长会保留 **30～60 分钟**。改对记录后如果依然不通过，**不要反复删改记录**，耐心等待 30～60 分钟缓存过期后再点一次提交即可通过。

---

## 第二步：切换自定义 NS 服务器

HE.net 成功添加域名后，回到 Yunamo 面板切换到 **自定义 NS**，填入 HE.net 的 5 个名称服务器并保存：
- `ns1.he.net`
- `ns2.he.net`
- `ns3.he.net`
- `ns4.he.net`
- `ns5.he.net`

---

## 第三步：在 HE.net 配置 GitHub Pages 解析

因为没有自己的服务器 IP，这里使用 GitHub Pages 作为托管（注意：HE.net 不支持根域名的 CNAME 拉平，所以根域名必须用 **A 记录**，不能用 CNAME）。

进入 HE.net 对应域名的管理页面，注意 **HE.net 的 Name 栏不要填 `@`**：

1. **添加 4 条根域名 A 记录**（点击 `New A`）：
   - **Name**：全部填 `nodeai.loc.cc`（不能写成 `github pages` 等自定义名称，否则会变成子域名）
   - **IPv4 Address**：分别填入 GitHub Pages 官方 IP：
     - `185.199.108.153`
     - `185.199.109.153`
     - `185.199.110.153`
     - `185.199.111.153`
   - **TTL**：`300`

2. **添加 `www` 子域名 CNAME 记录**（点击 `New CNAME`）：
   - **Name**：`www`
   - **Hostname**：`<你的GitHub用户名>.github.io`
   - **TTL**：`300`

> **为什么要加 `www`？**
> 在 GitHub Pages 绑定根域名时，GitHub 会同时检查 `www` 子域名的 DNS。如果没配 `www`，页面会出现黄色警告 `www.nodeai.loc.cc is improperly configured (InvalidDNSError)`。

---

## 第四步：在 GitHub 开启 Pages 并绑定域名

1. 在 GitHub 新建一个 **Public** 仓库，上传一个 `index.html`（或 `README.md`）。
2. 进入仓库的 **Settings -> Pages**：
   - **Source** 选择 `Deploy from a branch`，分支选 `main`（`/ (root)`），点击 **Save**。
   - 在 **Custom domain** 输入 `nodeai.loc.cc` 并点击 **Save**。
3. 如果先点了保存、后补加的 `www` CNAME 记录，同样可能触发 HE.net 的 SOA 负缓存（**900 秒 / 15 分钟**）。等待 15 分钟后点击 **Check again** 即可通过检查。
4. DNS 检查变绿、证书自动签发完成后，勾选 **Enforce HTTPS**，大功告成！

---

## 常见排查小工具

遇到 DNS 迟迟不生效时，可以用浏览器直接访问公共 DoH 接口查看全球实际解析结果，判断是“没同步”还是“被负缓存了”：
- Google DNS：`https://dns.google/resolve?name=你的域名&type=A`
- Cloudflare DNS：`https://cloudflare-dns.com/dns-query?name=你的域名&type=TXT`（需带请求头 `Accept: application/dns-json`）
