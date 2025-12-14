# WHOIS 查询问题快速修复指南

## 🎯 问题概述

### nyc.mn TLD 问题
- **症状**: 查询 `example.nyc.mn` 返回错误 "只能查询一级域名，不支持二级域名查询"
- **原因**: `.nyc.mn` 未被添加到二级域名白名单中
- **影响**: 无法查询任何 `.nyc.mn` 域名

### de TLD 问题  
- **症状**: 查询 `example.de` 可能失败或返回不完整信息
- **原因**: 可能是 WhoisJSON API 不支持或 DENIC 的 GDPR 限制
- **影响**: 德国域名无法可靠查询

---

## 🔧 快速修复方案

### 方案 1: 添加 nyc.mn 支持（推荐）

#### 步骤 1: 修改验证逻辑

在 `_worker.js` 第 5307 行后添加对 `.nyc.mn` 的检测：

```javascript
// 原代码（第 5306-5307 行）
const isPpUa = lowerDomain.endsWith('.pp.ua');
const isDigitalPlat = lowerDomain.endsWith('.qzz.io') || 
                      lowerDomain.endsWith('.dpdns.org') || 
                      lowerDomain.endsWith('.us.kg') || 
                      lowerDomain.endsWith('.xx.kg');

// 添加新的检测（在第 5307 行后）
const isNycMn = lowerDomain.endsWith('.nyc.mn');
```

#### 步骤 2: 更新验证条件

将第 5309 行的验证条件修改为：

```javascript
// 原代码
if (dotCount !== 1 && !((isPpUa || isDigitalPlat) && dotCount === 2)) {

// 修改为
if (dotCount !== 1 && !((isPpUa || isDigitalPlat || isNycMn) && dotCount === 2)) {
```

#### 步骤 3: 添加解析器选择

在第 5317-5327 行的解析器选择逻辑中添加 `.nyc.mn` 的分支：

```javascript
let result;
if (isPpUa) {
  result = await queryPpUaWhois(domain);
} else if (isNycMn) {
  // 选项 A: 尝试使用 WhoisJSON（如果支持）
  result = await queryDomainWhois(domain);
  
  // 选项 B: 实现专用解析器（见下方）
  // result = await queryNycMnWhois(domain);
} else if (isDigitalPlat) {
  result = await queryDigitalPlatWhois(domain);
} else {
  result = await queryDomainWhois(domain);
}
```

#### 步骤 4: （可选）实现专用解析器

如果 WhoisJSON 不支持 `.nyc.mn`，在 `queryDigitalPlatWhois` 函数后（第 241 行后）添加：

```javascript
// 查询 .nyc.mn 域名的 WHOIS 信息
async function queryNycMnWhois(domain) {
  try {
    // 使用蒙古 NIC 的 WHOIS 查询接口
    // 注意: 这需要找到合适的 API 或实现 WHOIS 协议客户端
    
    // 临时方案: 使用第三方 WHOIS API
    // 示例使用 WHOIS XML API（需要 API key）
    const apiKey = typeof WHOISXMLAPI_KEY !== 'undefined' ? WHOISXMLAPI_KEY : '';
    
    if (!apiKey) {
      throw new Error('WHOIS API密钥未配置');
    }
    
    const response = await fetch(
      `https://www.whoisxmlapi.com/whoisserver/WhoisService?` +
      `apiKey=${apiKey}&domainName=${encodeURIComponent(domain)}&outputFormat=JSON`,
      {
        method: 'GET',
        headers: {
          'accept': 'application/json',
          'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
        }
      }
    );
    
    if (!response.ok) {
      throw new Error(`API请求失败: ${response.status} ${response.statusText}`);
    }
    
    const data = await response.json();
    
    // 检查域名是否已注册
    if (!data.WhoisRecord || data.WhoisRecord.dataError) {
      return {
        success: true,
        domain: domain,
        registered: false,
        raw: data
      };
    }
    
    const record = data.WhoisRecord;
    
    return {
      success: true,
      domain: domain,
      registered: true,
      registrationDate: record.registryData?.createdDate ? formatDate(record.registryData.createdDate) : null,
      expiryDate: record.registryData?.expiresDate ? formatDate(record.registryData.expiresDate) : null,
      lastUpdated: record.registryData?.updatedDate ? formatDate(record.registryData.updatedDate) : null,
      registrar: record.registrarName || 'Datacom LLC',
      registrarUrl: 'https://www.nic.mn',
      nameservers: record.nameServers?.hostNames || [],
      status: record.status || [],
      dnssec: record.registryData?.dnssec || null,
      raw: data
    };
  } catch (error) {
    return {
      success: false,
      error: error.message,
      domain: domain
    };
  }
}
```

---

### 方案 2: 处理 de TLD

#### 步骤 1: 测试 WhoisJSON 支持

首先验证 WhoisJSON 是否支持 `.de` 域名：

```bash
# 使用 curl 测试
curl -H "Authorization: Token=YOUR_API_KEY" \
  "https://whoisjson.com/api/v1/whois?domain=example.de"
```

#### 步骤 2a: 如果支持，无需修改

如果 WhoisJSON 返回完整的 WHOIS 信息，则现有代码已经可以工作。

#### 步骤 2b: 如果不支持，添加专用处理

在第 5317-5327 行的解析器选择逻辑中添加：

```javascript
let result;
if (isPpUa) {
  result = await queryPpUaWhois(domain);
} else if (isNycMn) {
  result = await queryNycMnWhois(domain);
} else if (isDigitalPlat) {
  result = await queryDigitalPlatWhois(domain);
} else if (lowerDomain.endsWith('.de')) {
  // 为 .de 域名使用专用处理
  result = await queryDenicWhois(domain);
} else {
  result = await queryDomainWhois(domain);
}
```

#### 步骤 3: 实现 DENIC 解析器（如果需要）

在 `queryNycMnWhois` 函数后添加：

```javascript
// 查询 .de 域名的 WHOIS 信息
async function queryDenicWhois(domain) {
  try {
    // DENIC 不提供公开的 REST API，需要使用第三方服务
    // 选项 1: WHOIS XML API
    const apiKey = typeof WHOISXMLAPI_KEY !== 'undefined' ? WHOISXMLAPI_KEY : '';
    
    if (!apiKey) {
      // 回退到 WhoisJSON
      return await queryDomainWhois(domain);
    }
    
    const response = await fetch(
      `https://www.whoisxmlapi.com/whoisserver/WhoisService?` +
      `apiKey=${apiKey}&domainName=${encodeURIComponent(domain)}&outputFormat=JSON`,
      {
        method: 'GET',
        headers: {
          'accept': 'application/json'
        }
      }
    );
    
    if (!response.ok) {
      // 失败时回退到 WhoisJSON
      return await queryDomainWhois(domain);
    }
    
    const data = await response.json();
    
    if (!data.WhoisRecord || data.WhoisRecord.dataError) {
      return {
        success: true,
        domain: domain,
        registered: false,
        raw: data
      };
    }
    
    const record = data.WhoisRecord;
    
    return {
      success: true,
      domain: domain,
      registered: true,
      registrationDate: record.registryData?.createdDate ? formatDate(record.registryData.createdDate) : null,
      expiryDate: record.registryData?.expiresDate ? formatDate(record.registryData.expiresDate) : null,
      lastUpdated: record.registryData?.updatedDate ? formatDate(record.registryData.updatedDate) : null,
      registrar: record.registrarName || null,
      registrarUrl: record.registrarUrl || null,
      nameservers: record.nameServers?.hostNames || [],
      status: record.status || [],
      dnssec: record.registryData?.dnssec || null,
      raw: data,
      notice: 'Some information may be redacted due to GDPR compliance'
    };
  } catch (error) {
    // 出错时尝试回退到 WhoisJSON
    try {
      return await queryDomainWhois(domain);
    } catch {
      return {
        success: false,
        error: error.message,
        domain: domain
      };
    }
  }
}
```

---

## 🧪 测试清单

### 测试 nyc.mn 支持

```bash
# 测试 1: 查询已注册的 .nyc.mn 域名
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.nyc.mn"}'

# 预期: 返回 WHOIS 信息或 registered: false

# 测试 2: 验证错误不再出现
# 不应返回 "只能查询一级域名，不支持二级域名查询"
```

### 测试 de 支持

```bash
# 测试 1: 查询已知的 .de 域名
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.de"}'

# 预期: 返回 WHOIS 信息（可能部分字段因 GDPR 被隐藏）

# 测试 2: 查询未注册的 .de 域名
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"thisdomaindoesnotexist12345.de"}'

# 预期: registered: false
```

### 回归测试

确保现有功能不受影响：

```bash
# 测试 .pp.ua
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.pp.ua"}'

# 测试 .us.kg
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.us.kg"}'

# 测试 .com
curl -X POST https://your-worker.workers.dev/api/whois \
  -H "Cookie: auth=true" \
  -H "Content-Type: application/json" \
  -d '{"domain":"example.com"}'
```

---

## 📝 环境变量配置

如果使用额外的 WHOIS API 服务，需要添加环境变量：

### Cloudflare Workers 配置

```bash
# 在 wrangler.toml 中添加（开发环境）
[vars]
WHOISJSON_API_KEY = "your_whoisjson_key"
WHOISXMLAPI_KEY = "your_whoisxml_key"  # 如果使用

# 或在 Cloudflare Dashboard 中设置（生产环境）
# Workers & Pages > your-worker > Settings > Variables
```

### 环境变量说明

| 变量名 | 必需 | 用途 | 获取方式 |
|--------|------|------|---------|
| WHOISJSON_API_KEY | 是 | WhoisJSON API 认证 | https://whoisjson.com |
| WHOISXMLAPI_KEY | 否 | WHOIS XML API 认证（备用） | https://whoisxmlapi.com |

---

## ⚠️ 注意事项

### 1. API 配额限制
- WhoisJSON: 根据订阅计划
- WHOIS XML API: 免费 500 次/月

### 2. GDPR 影响
- `.de` 和其他欧洲域名可能隐藏部分信息
- 联系信息通常被屏蔽
- 仅提供基本的注册和到期信息

### 3. 查询频率
- 避免频繁查询同一域名
- 建议实现缓存机制
- 注意 API 服务商的速率限制

### 4. 错误处理
- 始终检查 `success` 字段
- 提供友好的错误提示
- 记录失败的查询以便调试

---

## 🚀 部署步骤

### 1. 备份现有代码
```bash
git add .
git commit -m "Backup before WHOIS changes"
```

### 2. 应用修改
- 按照上述方案修改 `_worker.js`
- 确保所有语法正确

### 3. 本地测试
```bash
wrangler dev
# 在另一个终端测试 API
```

### 4. 部署到生产
```bash
wrangler publish
```

### 5. 验证部署
- 测试 `.nyc.mn` 域名查询
- 测试 `.de` 域名查询
- 运行回归测试

---

## 🔍 故障排查

### 问题 1: "WhoisJSON API密钥未配置"
**解决**: 
```bash
wrangler secret put WHOISJSON_API_KEY
# 输入你的 API key
```

### 问题 2: ".nyc.mn 仍然报错"
**检查**:
1. 确认 `isNycMn` 变量已定义
2. 确认验证条件已更新
3. 清除浏览器缓存和 Worker 缓存

### 问题 3: ".de 查询失败"
**诊断**:
1. 检查 WhoisJSON 是否支持 `.de`
2. 验证 API key 是否有效
3. 查看错误日志了解具体原因

### 问题 4: "回退解析器不工作"
**检查**:
1. 确认 try-catch 块正确
2. 验证备用 API key 已配置
3. 测试每个解析器单独工作

---

## 📚 相关资源

- [WHOIS Analysis 完整报告](./WHOIS_ANALYSIS.md)
- [WhoisJSON 文档](https://whoisjson.com/docs)
- [WHOIS XML API 文档](https://www.whoisxmlapi.com/documentation)
- [DENIC WHOIS 指南](https://www.denic.de/en/service/whois-service/)
- [蒙古 NIC](https://www.nic.mn/)

---

**最后更新**: 2025年  
**版本**: 1.0  
**维护**: AI 代码助手
