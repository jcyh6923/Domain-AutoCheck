# Domain-AutoCheck WHOIS 查询逻辑分析报告

## 📋 目录
1. [WHOIS 查询入口点](#1-whois-查询入口点)
2. [TLD 支持机制](#2-tld-支持机制)
3. [解析器选择逻辑](#3-解析器选择逻辑)
4. [TLD 特殊处理](#4-tld-特殊处理)
5. [当前支持的 TLD 列表](#5-当前支持的-tld-列表)
6. [nyc.mn 和 de TLD 问题分析](#6-nycmn-和-de-tld-问题分析)
7. [限制条件和障碍](#7-限制条件和障碍)
8. [改进建议](#8-改进建议)

---

## 1. WHOIS 查询入口点

### 端点位置
- **文件**: `_worker.js`
- **行数**: 5289-5333
- **路径**: `/api/whois`
- **方法**: POST
- **认证**: 需要（通过 cookie 验证）

### 请求流程
```
用户请求 → handleRequest() → 认证检查 → handleApiRequest() → /api/whois 处理器
```

### 请求格式
```json
POST /api/whois
{
  "domain": "example.com"
}
```

### 响应格式（成功）
```json
{
  "success": true,
  "domain": "example.com",
  "registered": true,
  "registrationDate": "2023-01-01",
  "expiryDate": "2024-01-01",
  "lastUpdated": "2023-06-01",
  "registrar": "Example Registrar",
  "registrarUrl": "https://example-registrar.com",
  "nameservers": ["ns1.example.com", "ns2.example.com"],
  "status": ["clientTransferProhibited"],
  "dnssec": "unsigned",
  "raw": { /* 原始 API 响应数据 */ }
}
```

### 响应格式（失败）
```json
{
  "success": false,
  "error": "错误信息",
  "domain": "example.com"
}
```

---

## 2. TLD 支持机制

### 实现方式
系统采用**硬编码白名单**的方式判断 TLD 是否被支持。

### 代码位置 (第 5303-5315 行)
```javascript
// 验证是否为一级域名（只能有一个点），pp.ua及DigitalPlat特定域名除外
const dotCount = domain.split('.').length - 1;
const lowerDomain = domain.toLowerCase();
const isPpUa = lowerDomain.endsWith('.pp.ua');
const isDigitalPlat = lowerDomain.endsWith('.qzz.io') || 
                      lowerDomain.endsWith('.dpdns.org') || 
                      lowerDomain.endsWith('.us.kg') || 
                      lowerDomain.endsWith('.xx.kg');

if (dotCount !== 1 && !((isPpUa || isDigitalPlat) && dotCount === 2)) {
  if (dotCount === 0) {
    return jsonResponse({ error: '请输入完整的域名（如：example.com）' }, 400);
  } else {
    return jsonResponse({ error: '只能查询一级域名，不支持二级域名查询' }, 400);
  }
}
```

### 判断逻辑
1. **计算点数**: 通过 `domain.split('.').length - 1` 计算域名中的点数
2. **特殊域名检测**: 检查是否为 `.pp.ua` 或 DigitalPlat 域名
3. **验证规则**:
   - 普通域名：必须恰好有 1 个点（如 `example.com`）
   - 特殊域名：允许有 2 个点（如 `xxx.pp.ua`, `xxx.us.kg`）
   - 其他情况：拒绝查询

---

## 3. 解析器选择逻辑

系统提供三个 WHOIS 解析器，根据域名后缀自动选择：

### 3.1 queryPpUaWhois (第 104-168 行)

**适用域名**: `*.pp.ua`

**API 端点**: `https://nic.ua/en/whois-info?domain_name={domain}`

**特点**:
- 专门用于查询乌克兰 .pp.ua 二级域名
- 解析 NIC.UA 的 JSON 响应，提取文本格式的 WHOIS 信息
- 使用正则表达式解析字段：
  - Created On (注册日期)
  - Expiration Date (到期日期)
  - Last Updated On (更新日期)
  - Sponsoring Registrar (注册商)
  - Name Server (域名服务器)

**示例代码片段**:
```javascript
const response = await fetch(`https://nic.ua/en/whois-info?domain_name=${encodeURIComponent(domain)}`, {
  method: 'GET',
  headers: {
    'accept': '*/*',
    'user-agent': 'Mozilla/5.0 ...',
    'x-requested-with': 'XMLHttpRequest'
  }
});
```

### 3.2 queryDigitalPlatWhois (第 171-241 行)

**适用域名**: `*.qzz.io`, `*.dpdns.org`, `*.us.kg`, `*.xx.kg`

**API 端点**: `https://dash.domain.digitalplat.org/whois?name={domain}` (通过 corsproxy.io 代理)

**特点**:
- 用于 DigitalPlat 提供的免费二级域名服务
- 使用 CORS 代理避免 Cloudflare Workers 的 TLS/SSL 握手问题
- 解析纯文本 WHOIS 响应
- 支持 "Domain not found" 检测（域名未注册）

**代理使用**:
```javascript
const targetUrl = `https://dash.domain.digitalplat.org/whois?name=${encodeURIComponent(domain)}`;
const response = await fetch(`https://corsproxy.io/?url=${encodeURIComponent(targetUrl)}`, {
  method: 'GET',
  headers: {
    'accept': 'text/html',
    'user-agent': 'Mozilla/5.0 ...'
  }
});
```

### 3.3 queryDomainWhois (第 55-101 行)

**适用域名**: 所有其他一级域名（默认解析器）

**API 端点**: `https://whoisjson.com/api/v1/whois?domain={domain}`

**特点**:
- 使用 WhoisJSON 商业 API 服务
- **需要 API 密钥** (环境变量 `WHOISJSON_API_KEY`)
- 返回结构化的 JSON 数据，无需解析
- 支持大多数常见 TLD，具体取决于 WhoisJSON 服务支持范围

**配置要求**:
```javascript
const apiKey = typeof WHOISJSON_API_KEY !== 'undefined' ? WHOISJSON_API_KEY : DEFAULT_WHOISJSON_API_KEY;
if (!apiKey) {
  throw new Error('WhoisJSON API密钥未配置');
}
```

### 3.4 选择决策树

```
接收到域名
    ↓
是否以 .pp.ua 结尾?
    ├─ 是 → 使用 queryPpUaWhois (NIC.UA API)
    └─ 否 ↓
       是否以 .qzz.io/.dpdns.org/.us.kg/.xx.kg 结尾?
           ├─ 是 → 使用 queryDigitalPlatWhois (DigitalPlat API)
           └─ 否 → 使用 queryDomainWhois (WhoisJSON API)
```

**代码实现 (第 5317-5327 行)**:
```javascript
let result;
if (isPpUa) {
  // 使用专门的 nic.ua 接口查询 pp.ua 域名
  result = await queryPpUaWhois(domain);
} else if (isDigitalPlat) {
  // 使用 DigitalPlat 接口查询特定二级域名
  result = await queryDigitalPlatWhois(domain);
} else {
  // 其他域名使用 WhoisJSON API
  result = await queryDomainWhois(domain);
}
```

---

## 4. TLD 特殊处理

### 4.1 二级域名特殊支持

系统对以下二级域名提供特殊支持（允许 2 个点）：

#### pp.ua 域名
- **示例**: `example.pp.ua`
- **注册商**: NIC.UA (乌克兰网络信息中心)
- **特殊处理**: 使用 NIC.UA 专用 WHOIS API
- **原因**: `.pp.ua` 是乌克兰的免费二级域名服务，需要专门的查询接口

#### DigitalPlat 域名组
- **域名后缀**: 
  - `.qzz.io` - DigitalPlat 提供的免费域名
  - `.dpdns.org` - DigitalPlat DNS 服务域名
  - `.us.kg` - 基里巴斯 .kg 的二级域名
  - `.xx.kg` - 基里巴斯 .kg 的另一个二级域名
- **特殊处理**: 使用 DigitalPlat 的 WHOIS 服务，通过 CORS 代理访问
- **原因**: 这些是免费域名服务，不在标准 WHOIS 系统中

### 4.2 普通 TLD 处理

所有其他 TLD（如 `.com`, `.net`, `.org`, `.io` 等）：
- 必须是一级域名格式（只有一个点）
- 通过 WhoisJSON API 查询
- 支持范围取决于 WhoisJSON 服务

### 4.3 不支持的域名格式

**三级域名** (如 `sub.example.com`):
```
错误信息: "只能查询一级域名，不支持二级域名查询"
```

**顶级域名** (如 `com`):
```
错误信息: "请输入完整的域名（如：example.com）"
```

---

## 5. 当前支持的 TLD 列表

### 5.1 明确支持的 TLD（代码级别）

| TLD 后缀 | 类型 | 解析器 | 说明 |
|---------|------|--------|------|
| `.pp.ua` | 二级域名 | queryPpUaWhois | 乌克兰免费二级域名 |
| `.qzz.io` | 二级域名 | queryDigitalPlatWhois | DigitalPlat 提供 |
| `.dpdns.org` | 二级域名 | queryDigitalPlatWhois | DigitalPlat DNS 服务 |
| `.us.kg` | 二级域名 | queryDigitalPlatWhois | 基里巴斯二级域名 |
| `.xx.kg` | 二级域名 | queryDigitalPlatWhois | 基里巴斯二级域名 |

### 5.2 理论支持的 TLD（通过 WhoisJSON API）

WhoisJSON API 支持的常见 TLD 包括但不限于：

**通用顶级域名 (gTLD)**:
- `.com`, `.net`, `.org`, `.info`, `.biz`
- `.name`, `.pro`, `.mobi`, `.tel`, `.asia`

**新通用顶级域名 (new gTLD)**:
- `.io`, `.ai`, `.co`, `.app`, `.dev`
- `.cloud`, `.online`, `.site`, `.store`, `.tech`
- `.blog`, `.shop`, `.news`, `.email`, `.digital`

**国家和地区顶级域名 (ccTLD)**:
- `.us`, `.uk`, `.ca`, `.au`, `.jp`
- `.cn`, `.in`, `.br`, `.ru`, `.fr`
- `.nl`, `.it`, `.es`, `.se`, `.no`
- `.ch`, `.at`, `.be`, `.pl`, `.cz`

> **注意**: 
> 1. WhoisJSON 的 TLD 支持范围需要查看其官方文档
> 2. 某些 ccTLD 可能有查询限制或需要特殊权限
> 3. 未列出的 TLD 不代表不支持，需要实际测试

### 5.3 不支持或有问题的 TLD

| TLD | 状态 | 原因 |
|-----|------|------|
| `.nyc.mn` | ❌ 不支持 | 二级域名未加入白名单 |
| `.de` | ⚠️ 可能不支持 | 取决于 WhoisJSON 是否支持 DENIC |
| `.eu` | ⚠️ 可能有限制 | 欧盟域名有隐私保护 |
| 其他二级域名 | ❌ 不支持 | 除白名单外的二级域名被拒绝 |

---

## 6. nyc.mn 和 de TLD 问题分析

### 6.1 nyc.mn 域名问题

#### 问题表现
```
请求: { "domain": "example.nyc.mn" }
响应: { "error": "只能查询一级域名，不支持二级域名查询" }
```

#### 根本原因
1. `.nyc.mn` 是蒙古 `.mn` TLD 下的二级域名
2. 域名 `example.nyc.mn` 包含 2 个点
3. 代码在第 5306-5307 行检查特殊域名时，**没有包含 `.nyc.mn`**：
   ```javascript
   const isPpUa = lowerDomain.endsWith('.pp.ua');
   const isDigitalPlat = lowerDomain.endsWith('.qzz.io') || 
                         lowerDomain.endsWith('.dpdns.org') || 
                         lowerDomain.endsWith('.us.kg') || 
                         lowerDomain.endsWith('.xx.kg');
   // ↑ 缺少 .nyc.mn 的检测
   ```
4. 验证逻辑（第 5309 行）判定为无效的二级域名，直接拒绝

#### 技术背景
- `.nyc.mn` 是蒙古域名注册机构 Datacom 提供的免费二级域名服务
- WHOIS 服务器: `whois.nic.mn`
- 注册商网站: https://www.nic.mn/

#### 详细调用流程
```
1. 用户输入 "example.nyc.mn"
2. dotCount = 2 (两个点)
3. isPpUa = false (不是 .pp.ua)
4. isDigitalPlat = false (不在白名单中)
5. 条件判断: dotCount !== 1 && !((isPpUa || isDigitalPlat) && dotCount === 2)
   → 2 !== 1 && !((false || false) && 2 === 2)
   → true && !(false && true)
   → true && !false
   → true && true
   → true (触发错误)
6. 返回错误: "只能查询一级域名，不支持二级域名查询"
```

### 6.2 de TLD 问题

#### 问题表现
```
请求: { "domain": "example.de" }
可能的响应:
  1. { "error": "WhoisJSON API密钥未配置" }
  2. { "error": "API请求失败: 4xx/5xx" }
  3. { "success": false, "error": "不支持的 TLD" }
```

#### 根本原因

**原因 1: WhoisJSON API 限制**
- `.de` 域名由 DENIC (德国网络信息中心) 管理
- DENIC 有严格的 WHOIS 查询限制和隐私保护
- WhoisJSON API 可能：
  - 不支持 `.de` TLD
  - 需要特殊订阅计划
  - 查询受到速率限制

**原因 2: GDPR 隐私保护**
- 德国严格执行 GDPR (通用数据保护条例)
- `.de` 域名的 WHOIS 信息大量被隐藏
- 即使查询成功，返回的信息也可能不完整

**原因 3: DENIC 的特殊要求**
- DENIC WHOIS 服务器: `whois.denic.de`
- 需要使用特定的查询格式
- 可能需要通过 DENIC 的 Web 接口查询

#### 技术背景
- DENIC 官网: https://www.denic.de/
- WHOIS 服务: `whois -h whois.denic.de example.de`
- Web WHOIS: https://www.denic.de/webwhois/

#### 详细调用流程
```
1. 用户输入 "example.de"
2. dotCount = 1 (一个点)
3. isPpUa = false
4. isDigitalPlat = false
5. 验证通过，进入 WhoisJSON 查询分支
6. 调用 queryDomainWhois("example.de")
7. 检查 WHOISJSON_API_KEY:
   - 如果未配置 → 返回错误 "WhoisJSON API密钥未配置"
   - 如果已配置 → 继续
8. 发送请求到 WhoisJSON API
9. 可能的结果:
   a) API 返回 4xx 错误 (TLD 不支持)
   b) API 返回 200 但数据不完整 (隐私保护)
   c) API 返回 5xx 错误 (服务器问题)
```

### 6.3 问题对比总结

| 特征 | nyc.mn | de |
|------|--------|-----|
| **域名级别** | 二级域名 | 一级域名 |
| **点数** | 2 | 1 |
| **验证阶段失败** | ✅ 是 (代码验证) | ❌ 否 |
| **到达解析器** | ❌ 否 | ✅ 是 |
| **失败原因** | 未加入白名单 | API/注册局限制 |
| **错误阶段** | 请求验证阶段 | WHOIS 查询阶段 |
| **修复难度** | 容易 | 中等 |

---

## 7. 限制条件和障碍

### 7.1 架构限制

#### 硬编码白名单
- **问题**: 支持的二级域名写死在代码中
- **影响**: 添加新 TLD 需要修改代码并重新部署
- **维护成本**: 每次新增 TLD 都需要开发介入

#### 单一验证逻辑
- **问题**: 所有域名使用相同的点数验证
- **影响**: 无法灵活支持不同结构的域名（如 `.co.uk`, `.com.cn`）
- **扩展性**: 难以适应复杂的域名体系

#### 缺少配置系统
- **问题**: 没有 TLD 配置数据库或配置文件
- **影响**: 不能动态添加/移除 TLD 支持
- **灵活性**: 无法根据 API 服务变化快速调整

### 7.2 解析器限制

#### WhoisJSON API 依赖
- **单点故障**: 大部分 TLD 依赖一个商业 API
- **成本问题**: API 调用可能有配额和费用限制
- **支持范围**: 受限于 WhoisJSON 支持的 TLD 列表
- **服务可用性**: API 服务中断将影响所有非白名单 TLD

#### 缺少回退机制
- **问题**: 没有备用解析器或故障转移
- **影响**: 某个解析器失败后无法尝试其他方式
- **可靠性**: 降低了系统的容错能力

#### 解析器不通用
- **问题**: 每个解析器只能处理特定 TLD
- **影响**: 无法共享解析逻辑，代码重复
- **扩展性**: 添加新 TLD 可能需要新的解析器

### 7.3 功能限制

#### 不支持复杂域名结构
- **多级域名**: 如 `example.co.uk` (英国), `example.com.cn` (中国)
- **特殊后缀**: 如 `.gov.uk`, `.ac.uk`, `.edu.cn`
- **免费子域名**: 除白名单外的其他二级域名服务

#### 缺少 TLD 发现机制
- **问题**: 用户无法查询哪些 TLD 被支持
- **影响**: 只能通过试错来确定是否支持
- **用户体验**: 容易引起困惑和挫败感

#### 错误信息不够具体
```javascript
// 当前错误信息
{ "error": "只能查询一级域名，不支持二级域名查询" }

// 理想的错误信息应该是:
{
  "error": "不支持的域名格式或 TLD",
  "domain": "example.nyc.mn",
  "reason": "二级域名 .nyc.mn 未被支持",
  "supportedSecondLevel": [".pp.ua", ".qzz.io", ".dpdns.org", ".us.kg", ".xx.kg"],
  "suggestion": "如需支持此 TLD，请联系管理员"
}
```

### 7.4 技术障碍

#### CORS 和 TLS 问题
- **Cloudflare Workers 限制**: 某些 WHOIS 服务器无法直接访问
- **解决方案**: 使用 CORS 代理（如 DigitalPlat 的实现）
- **新问题**: 增加了延迟，引入了额外的依赖

#### WHOIS 协议复杂性
- **文本格式不统一**: 不同注册局的 WHOIS 输出格式各异
- **解析难度**: 需要为每种格式编写解析器
- **维护成本**: 注册局更新格式后需要更新解析器

#### 隐私和访问限制
- **GDPR**: 欧洲域名信息被大量隐藏
- **速率限制**: 某些注册局限制查询频率
- **认证要求**: 部分注册局需要账户或 API 密钥

### 7.5 扩展障碍

#### 添加新 TLD 的步骤复杂
1. 确定 TLD 类型（一级/二级）
2. 找到对应的 WHOIS 服务器或 API
3. 修改代码验证逻辑
4. 实现或选择合适的解析器
5. 编写解析规则
6. 测试并部署
7. 更新文档

这个过程耗时且容易出错，不适合快速添加支持。

#### 缺少自动化测试
- **问题**: 没有 TLD 支持的自动化测试套件
- **影响**: 难以确保修改不会破坏现有功能
- **维护**: 回归测试需要手动进行

---

## 8. 改进建议

### 8.1 针对 nyc.mn 的即时解决方案

#### 方案 1: 添加到白名单（最简单）

**修改代码** (第 5307 行):
```javascript
// 修改前
const isDigitalPlat = lowerDomain.endsWith('.qzz.io') || 
                      lowerDomain.endsWith('.dpdns.org') || 
                      lowerDomain.endsWith('.us.kg') || 
                      lowerDomain.endsWith('.xx.kg');

// 修改后
const isDigitalPlat = lowerDomain.endsWith('.qzz.io') || 
                      lowerDomain.endsWith('.dpdns.org') || 
                      lowerDomain.endsWith('.us.kg') || 
                      lowerDomain.endsWith('.xx.kg') ||
                      lowerDomain.endsWith('.nyc.mn');  // 新增
```

**优点**:
- 修改简单，影响最小
- 立即可用

**缺点**:
- 需要实现 `.nyc.mn` 的解析器
- 可能需要找到合适的 WHOIS API

#### 方案 2: 实现专用解析器

**新增函数**:
```javascript
// 查询 .nyc.mn 域名的 WHOIS 信息
async function queryNycMnWhois(domain) {
  try {
    // 方案 A: 使用蒙古 NIC 的 WHOIS 服务
    // whois.nic.mn 可能有限制，需要测试
    
    // 方案 B: 使用第三方 WHOIS API
    const response = await fetch(`https://www.whoisxmlapi.com/whoisserver/WhoisService?apiKey=${API_KEY}&domainName=${domain}&outputFormat=JSON`);
    
    // 方案 C: 使用 Datacom 的 Web 界面（如果有 API）
    // https://www.nic.mn/whois/lookup
    
    if (!response.ok) {
      throw new Error(`查询失败: ${response.status}`);
    }
    
    const data = await response.json();
    
    return {
      success: true,
      domain: domain,
      registered: true,
      registrationDate: data.registryData?.createdDate ? formatDate(data.registryData.createdDate) : null,
      expiryDate: data.registryData?.expiresDate ? formatDate(data.registryData.expiresDate) : null,
      registrar: data.registrarName || 'Datacom LLC',
      registrarUrl: 'https://www.nic.mn',
      // ... 其他字段
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

**调用逻辑修改** (第 5317-5327 行):
```javascript
let result;
if (isPpUa) {
  result = await queryPpUaWhois(domain);
} else if (lowerDomain.endsWith('.nyc.mn')) {  // 新增
  result = await queryNycMnWhois(domain);
} else if (isDigitalPlat) {
  result = await queryDigitalPlatWhois(domain);
} else {
  result = await queryDomainWhois(domain);
}
```

### 8.2 针对 de TLD 的解决方案

#### 方案 1: 验证 WhoisJSON 支持

**测试步骤**:
1. 使用 WhoisJSON API 测试查询 `.de` 域名
2. 检查返回结果的完整性
3. 评估是否满足需求

**代码无需修改**，但需要:
- 确认 API 订阅计划包含 `.de`
- 测试查询速率和准确性
- 了解返回数据的限制（GDPR 影响）

#### 方案 2: 使用 DENIC 官方接口

**新增函数**:
```javascript
async function queryDenicWhois(domain) {
  try {
    // DENIC 不提供 REST API，需要使用 WHOIS 协议
    // 在 Cloudflare Workers 中实现 WHOIS 协议客户端
    // 或使用第三方 WHOIS 网关服务
    
    // 选项 A: 使用 WHOIS 网关服务
    const response = await fetch(`https://api.whoapi.com/?domain=${domain}&apikey=${API_KEY}&r=whois`);
    
    if (!response.ok) {
      throw new Error(`DENIC 查询失败: ${response.status}`);
    }
    
    const data = await response.json();
    
    // 解析 DENIC 格式的 WHOIS 数据
    return {
      success: true,
      domain: domain,
      registered: data.status !== 'free',
      registrationDate: data.date_created ? formatDate(data.date_created) : null,
      expiryDate: data.date_expires ? formatDate(data.date_expires) : null,
      registrar: data.registrar_name || null,
      // 注意: GDPR 可能导致某些字段为空
      // ...
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

**修改路由逻辑**:
```javascript
let result;
if (isPpUa) {
  result = await queryPpUaWhois(domain);
} else if (isDigitalPlat) {
  result = await queryDigitalPlatWhois(domain);
} else if (lowerDomain.endsWith('.de')) {  // 新增
  result = await queryDenicWhois(domain);
} else {
  result = await queryDomainWhois(domain);
}
```

#### 方案 3: 多 API 回退策略

```javascript
async function queryWithFallback(domain) {
  // 尝试 API 1: WhoisJSON
  try {
    const result = await queryDomainWhois(domain);
    if (result.success) return result;
  } catch (error) {
    console.error('WhoisJSON 失败:', error);
  }
  
  // 尝试 API 2: WhoAPI
  try {
    const result = await queryWhoApiWhois(domain);
    if (result.success) return result;
  } catch (error) {
    console.error('WhoAPI 失败:', error);
  }
  
  // 尝试 API 3: WHOIS XML API
  try {
    const result = await queryWhoisXmlApi(domain);
    if (result.success) return result;
  } catch (error) {
    console.error('WHOIS XML API 失败:', error);
  }
  
  // 所有尝试都失败
  return {
    success: false,
    error: '无法查询该域名的 WHOIS 信息',
    domain: domain
  };
}
```

### 8.3 长期架构改进建议

#### 改进 1: 实现 TLD 配置系统

**创建配置结构** (存储在 KV 或 JSON 文件):
```json
{
  "tlds": [
    {
      "suffix": ".pp.ua",
      "type": "second-level",
      "parser": "queryPpUaWhois",
      "priority": 1,
      "enabled": true,
      "metadata": {
        "registry": "NIC.UA",
        "country": "Ukraine",
        "whoisServer": "whois.nic.ua"
      }
    },
    {
      "suffix": ".nyc.mn",
      "type": "second-level",
      "parser": "queryNycMnWhois",
      "priority": 1,
      "enabled": true,
      "metadata": {
        "registry": "Datacom LLC",
        "country": "Mongolia",
        "whoisServer": "whois.nic.mn"
      }
    },
    {
      "suffix": ".de",
      "type": "top-level",
      "parser": "queryDenicWhois",
      "priority": 2,
      "fallback": "queryDomainWhois",
      "enabled": true,
      "metadata": {
        "registry": "DENIC",
        "country": "Germany",
        "whoisServer": "whois.denic.de",
        "notes": "GDPR restrictions apply"
      }
    },
    {
      "suffix": "*",
      "type": "top-level",
      "parser": "queryDomainWhois",
      "priority": 999,
      "enabled": true,
      "metadata": {
        "description": "Default parser for all other TLDs"
      }
    }
  ],
  "parsers": {
    "queryPpUaWhois": {
      "endpoint": "https://nic.ua/en/whois-info",
      "method": "GET",
      "requiresAuth": false
    },
    "queryDomainWhois": {
      "endpoint": "https://whoisjson.com/api/v1/whois",
      "method": "GET",
      "requiresAuth": true,
      "authEnvVar": "WHOISJSON_API_KEY"
    }
  }
}
```

**实现配置驱动的路由**:
```javascript
async function getTldConfig(domain) {
  const config = await DOMAIN_MONITOR.get('tld_config');
  const tldConfigs = JSON.parse(config).tlds;
  
  // 按优先级排序
  tldConfigs.sort((a, b) => a.priority - b.priority);
  
  // 找到匹配的配置
  for (const tldConfig of tldConfigs) {
    if (tldConfig.suffix === '*' || domain.endsWith(tldConfig.suffix)) {
      return tldConfig;
    }
  }
  
  return null;
}

async function queryWhoisDynamic(domain) {
  const tldConfig = await getTldConfig(domain);
  
  if (!tldConfig || !tldConfig.enabled) {
    return {
      success: false,
      error: '不支持的 TLD',
      domain: domain
    };
  }
  
  // 根据配置调用相应的解析器
  const parser = parsers[tldConfig.parser];
  let result = await parser(domain);
  
  // 如果失败且有回退解析器，尝试回退
  if (!result.success && tldConfig.fallback) {
    const fallbackParser = parsers[tldConfig.fallback];
    result = await fallbackParser(domain);
  }
  
  return result;
}
```

#### 改进 2: 统一解析器接口

**定义标准接口**:
```javascript
class WhoisParser {
  constructor(config) {
    this.config = config;
  }
  
  // 所有解析器必须实现的方法
  async query(domain) {
    throw new Error('Must implement query method');
  }
  
  // 标准化响应格式
  formatResponse(rawData) {
    return {
      success: true,
      domain: rawData.domain,
      registered: rawData.registered,
      registrationDate: rawData.created ? formatDate(rawData.created) : null,
      expiryDate: rawData.expires ? formatDate(rawData.expires) : null,
      lastUpdated: rawData.updated ? formatDate(rawData.updated) : null,
      registrar: rawData.registrar?.name || null,
      registrarUrl: rawData.registrar?.url || null,
      nameservers: rawData.nameservers || [],
      status: rawData.status || [],
      dnssec: rawData.dnssec || null,
      raw: rawData
    };
  }
}

class WhoisJsonParser extends WhoisParser {
  async query(domain) {
    const apiKey = this.config.apiKey;
    const response = await fetch(`https://whoisjson.com/api/v1/whois?domain=${domain}`, {
      headers: { 'Authorization': `Token=${apiKey}` }
    });
    const data = await response.json();
    return this.formatResponse(data);
  }
}

class PpUaParser extends WhoisParser {
  async query(domain) {
    const response = await fetch(`https://nic.ua/en/whois-info?domain_name=${domain}`);
    const data = await response.json();
    // 解析 NIC.UA 格式
    return this.formatResponse(this.parseNicUa(data));
  }
  
  parseNicUa(data) {
    const whoisText = data.whois_info || '';
    // ... 解析逻辑
    return parsedData;
  }
}

// 使用
const parser = ParserFactory.create(tldConfig.parser, config);
const result = await parser.query(domain);
```

#### 改进 3: 添加 TLD 发现 API

**新增端点**: `/api/whois/supported-tlds`

```javascript
// 获取支持的 TLD 列表
if (path === '/api/whois/supported-tlds' && request.method === 'GET') {
  try {
    const config = await DOMAIN_MONITOR.get('tld_config');
    const tldConfigs = JSON.parse(config).tlds;
    
    const supportedTlds = tldConfigs
      .filter(tld => tld.enabled && tld.suffix !== '*')
      .map(tld => ({
        suffix: tld.suffix,
        type: tld.type,
        registry: tld.metadata?.registry,
        country: tld.metadata?.country,
        notes: tld.metadata?.notes
      }));
    
    return jsonResponse({
      success: true,
      tlds: supportedTlds,
      totalCount: supportedTlds.length
    });
  } catch (error) {
    return jsonResponse({ error: '获取 TLD 列表失败' }, 500);
  }
}

// 检查特定 TLD 是否支持
if (path === '/api/whois/check-tld' && request.method === 'POST') {
  try {
    const { domain } = await request.json();
    const tldConfig = await getTldConfig(domain);
    
    return jsonResponse({
      success: true,
      domain: domain,
      supported: !!tldConfig && tldConfig.enabled,
      parser: tldConfig?.parser,
      type: tldConfig?.type,
      metadata: tldConfig?.metadata
    });
  } catch (error) {
    return jsonResponse({ error: '检查失败' }, 500);
  }
}
```

#### 改进 4: 增强错误处理和提示

**改进错误响应**:
```javascript
function createDetailedError(domain, reason, suggestions = []) {
  // 提取 TLD
  const parts = domain.split('.');
  const tld = parts.length > 1 ? '.' + parts.slice(-1).join('.') : null;
  const secondLevelTld = parts.length > 2 ? '.' + parts.slice(-2).join('.') : null;
  
  return {
    success: false,
    error: reason,
    domain: domain,
    details: {
      tld: tld,
      secondLevelTld: secondLevelTld,
      detectedFormat: parts.length === 2 ? 'top-level' : 
                      parts.length === 3 ? 'second-level' : 'multi-level'
    },
    suggestions: suggestions,
    supportedExamples: {
      topLevel: ['example.com', 'example.net', 'example.io'],
      secondLevel: ['example.pp.ua', 'example.us.kg', 'example.qzz.io']
    }
  };
}

// 使用示例
if (dotCount !== 1 && !((isPpUa || isDigitalPlat) && dotCount === 2)) {
  if (dotCount === 0) {
    return jsonResponse(
      createDetailedError(
        domain, 
        '请输入完整的域名',
        ['添加顶级域名后缀，例如: ' + domain + '.com']
      ), 
      400
    );
  } else {
    // 检查是否可能是未支持的二级域名
    const parts = domain.split('.');
    const possibleSecondLevelTld = '.' + parts.slice(-2).join('.');
    
    return jsonResponse(
      createDetailedError(
        domain,
        '不支持的域名格式',
        [
          '如果 "' + possibleSecondLevelTld + '" 是二级域名服务，请联系管理员添加支持',
          '当前支持的二级域名: .pp.ua, .qzz.io, .dpdns.org, .us.kg, .xx.kg'
        ]
      ),
      400
    );
  }
}
```

#### 改进 5: 实现缓存机制

**减少 API 调用，提高性能**:
```javascript
async function queryCachedWhois(domain) {
  const cacheKey = `whois_cache_${domain}`;
  const cacheTTL = 3600; // 1小时
  
  // 尝试从缓存获取
  const cached = await DOMAIN_MONITOR.get(cacheKey);
  if (cached) {
    const data = JSON.parse(cached);
    const age = Date.now() - data.timestamp;
    
    if (age < cacheTTL * 1000) {
      return {
        ...data.result,
        cached: true,
        cacheAge: Math.floor(age / 1000)
      };
    }
  }
  
  // 缓存未命中或已过期，执行查询
  const result = await queryWhoisDynamic(domain);
  
  // 只缓存成功的查询
  if (result.success) {
    await DOMAIN_MONITOR.put(cacheKey, JSON.stringify({
      result: result,
      timestamp: Date.now()
    }), {
      expirationTtl: cacheTTL
    });
  }
  
  return result;
}
```

### 8.4 实施优先级

#### 🔥 高优先级（立即实施）
1. **添加 .nyc.mn 支持** - 快速解决用户需求
2. **验证 .de 支持** - 确认 WhoisJSON 是否可用
3. **改进错误信息** - 提升用户体验

#### ⚡ 中优先级（近期实施）
4. **实现 TLD 配置系统** - 提高扩展性
5. **添加 TLD 发现 API** - 让用户知道哪些 TLD 被支持
6. **实现基本缓存** - 减少 API 调用成本

#### 💡 低优先级（长期规划）
7. **统一解析器接口** - 代码重构，提高可维护性
8. **多 API 回退策略** - 提高可靠性
9. **自动化测试套件** - 确保质量

### 8.5 具体实施建议

#### 第一阶段：快速修复（1-2 天）

**目标**: 让 .nyc.mn 和 .de 可用

1. 添加 .nyc.mn 到白名单
2. 实现 queryNycMnWhois 函数（可以先使用 WhoisJSON 尝试）
3. 测试 .de 域名查询
4. 如果 .de 不可用，添加明确的错误提示

**测试清单**:
- [ ] example.nyc.mn 可以查询
- [ ] example.de 可以查询或返回明确错误
- [ ] 现有 TLD 功能不受影响

#### 第二阶段：架构优化（1-2 周）

**目标**: 建立可扩展的 TLD 管理系统

1. 设计 TLD 配置数据结构
2. 实现配置驱动的路由逻辑
3. 添加 TLD 管理 API（增删改查）
4. 创建 TLD 配置管理界面

**测试清单**:
- [ ] 可以通过 API 添加新 TLD
- [ ] 可以动态启用/禁用 TLD
- [ ] 配置持久化到 KV
- [ ] 后向兼容现有代码

#### 第三阶段：功能增强（2-4 周）

**目标**: 提升系统可靠性和用户体验

1. 实现多解析器回退
2. 添加 WHOIS 结果缓存
3. 创建 TLD 发现 API
4. 改进错误处理和提示
5. 添加查询历史和统计

**测试清单**:
- [ ] 解析器故障时能自动回退
- [ ] 缓存正确工作，减少 API 调用
- [ ] 用户可以查询支持的 TLD 列表
- [ ] 错误信息清晰且可操作

---

## 9. 总结

### 9.1 当前状态
- ✅ 支持 5 个明确的二级域名 TLD
- ✅ 通过 WhoisJSON 支持大多数常见 TLD
- ❌ 不支持 .nyc.mn（硬编码限制）
- ⚠️ .de 支持情况未知（需要测试）

### 9.2 主要问题
1. 硬编码的白名单导致扩展困难
2. 缺少配置系统和动态管理能力
3. 依赖单一 API 服务，缺少冗余
4. 错误信息不够友好和具体
5. 没有 TLD 发现和验证机制

### 9.3 推荐行动
1. **立即**: 添加 .nyc.mn 支持，验证 .de 可用性
2. **短期**: 实现 TLD 配置系统，添加发现 API
3. **中期**: 统一解析器接口，实现回退机制
4. **长期**: 建立自动化测试，持续优化用户体验

### 9.4 预期收益
- 🎯 用户可以查询更多 TLD 的域名
- 🚀 管理员可以快速添加新 TLD 支持
- 💪 系统更稳定，有故障转移能力
- 😊 用户体验更好，错误提示更清晰

---

## 附录

### A. 相关 API 服务

| 服务名称 | 官网 | 免费额度 | 特点 |
|---------|------|---------|------|
| WhoisJSON | https://whoisjson.com | ❌ | 支持 TLD 广泛 |
| WHOIS XML API | https://www.whoisxmlapi.com | ✅ 500次/月 | 数据详细 |
| WhoAPI | https://whoapi.com | ❌ | 响应快速 |
| WhoisFreaks | https://whoisfreaks.com | ✅ 10,000次/月 | 实时数据 |
| IP2WHOIS | https://www.ip2whois.com | ✅ 500次/月 | 简单易用 |

### B. 常见 WHOIS 服务器

| TLD | WHOIS 服务器 | 说明 |
|-----|-------------|------|
| .com | whois.verisign-grs.com | 商业域名 |
| .net | whois.verisign-grs.com | 网络域名 |
| .org | whois.pir.org | 组织域名 |
| .de | whois.denic.de | 德国域名 |
| .uk | whois.nic.uk | 英国域名 |
| .cn | whois.cnnic.cn | 中国域名 |
| .mn | whois.nic.mn | 蒙古域名 |
| .io | whois.nic.io | 技术域名 |

### C. 代码位置索引

| 功能 | 文件 | 行数 |
|------|------|------|
| WHOIS 端点处理 | _worker.js | 5289-5333 |
| WhoisJSON 解析器 | _worker.js | 55-101 |
| PP.UA 解析器 | _worker.js | 104-168 |
| DigitalPlat 解析器 | _worker.js | 171-241 |
| 域名验证逻辑 | _worker.js | 5297-5315 |
| 解析器选择逻辑 | _worker.js | 5317-5327 |

---

**文档版本**: 1.0  
**分析日期**: 2025年  
**分析对象**: Domain-AutoCheck WHOIS 查询系统  
**分析人**: AI 代码分析助手
