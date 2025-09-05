# 微信支付API v3升级说明文档

## 升级概述
本项目已完成从微信支付API v2到v3的前端升级，通过5次迭代优化，提升了代码的逻辑合理性、完整性和强壮性。

**重要提醒**：微信支付API v2将于2025年6月30日停止服务，建议尽快完成升级。

## 向后兼容性保证 🔄

### 兼容性问题解决
为了确保升级到微信支付 API v3 不会破坏现有功能，我们实现了完整的向后兼容性方案：

#### 主要改进
1. **智能参数检测**：自动识别 v2 (MD5) 和 v3 (RSA) 格式参数
2. **统一接口保持**：外部调用接口保持不变，内部自动适配
3. **渐进式升级**：支持 v2/v3 混合环境，无需一次性全部升级

#### 技术实现
```javascript
// 自动检测参数格式
const isV3Format = payData.appId && payData.signType === 'RSA';

// 统一处理方法
validateAndNormalizeWeChatPayParams(payData) {
    // 通用参数验证
    // v2/v3 特定验证
    // 返回标准化参数
}
```

#### 兼容性测试
- 详细测试指南：`docs/BACKWARD_COMPATIBILITY_TEST.md`
- 覆盖所有支付场景：商品支付、收银台支付、H5支付
- 确保零停机升级

## API v2 vs v3 核心差异对比

| 差异项 | API v2 | API v3 | 影响等级 |
|--------|--------|--------|----------|
| 数据格式 | XML | JSON | 🔴 高 |
| 签名算法 | MD5/HMAC-SHA256 | RSA/ECDSA | 🔴 高 |
| 证书要求 | 可选 | 强制 | 🔴 高 |
| 接口风格 | 传统API | RESTful | 🟡 中 |
| 回调验证 | 签名验证 | 证书+签名 | 🔴 高 |
| 前端调用 | signType可选 | signType必须RSA | 🟡 中 |

## 前端升级详情

### 关键参数变化
**API v2前端调用参数：**
```javascript
uni.requestPayment({
  timeStamp: "1640995200",
  nonceStr: "abc123",
  package: "prepay_id=wx123456...",
  signType: "MD5",  // 可以是MD5或HMAC-SHA256
  paySign: "C380BEC2BFD727A4B6845133519F3AD6"
});
```

**API v3前端调用参数：**
```javascript
uni.requestPayment({
  appId: "wx1234567890123456",  // v3新增appId参数
  timeStamp: "1640995200",
  nonceStr: "abc123",
  package: "prepay_id=wx123456...",
  signType: "RSA",  // v3必须是RSA
  paySign: "Base64编码的RSA签名"  // RSA签名算法生成
});
```

## 主要修改文件

### 1. `/components/payment/payment.vue` - 主支付组件
**修改位置：**
- 第588-650行：小程序支付处理
- 第674-740行：H5支付处理
- methods 新增验证和错误处理方法

### 2. `/pages/plugins/allocation/cashier/cashier.vue` - 收银台组件
**修改位置：**
- 第151-180行：支付处理方法
- methods 新增参数验证和错误处理方法

## 5次迭代优化详情

### 第1次迭代：基础优化
- **目标**：添加参数验证和基础错误处理
- **改进**：
  - 在支付调用前添加参数验证
  - 优化注释说明，明确API v3支持

### 第2次迭代：参数验证增强
- **目标**：完善参数验证逻辑
- **改进**：
  - 新增 `validateWeChatPayV3Params()` 方法
  - 新增 `handleWeChatPayV3Error()` 错误处理方法
  - 验证必要参数完整性
  - 验证时间戳、随机字符串、签名类型格式

### 第3次迭代：支付调用优化
- **目标**：优化支付调用和H5端处理
- **改进**：
  - H5端添加参数验证
  - 增强支付前后日志记录
  - 优化H5支付结果处理
  - 统一错误处理逻辑

### 第4次迭代：重试机制和状态管理
- **目标**：添加重试机制，提升支付成功率
- **改进**：
  - 新增支付状态管理（idle、processing、success、failed）
  - 新增重试机制（最多3次，递增延迟）
  - 新增 `shouldRetryPayment()` 判断可重试错误
  - 新增 `executeWeChatPayV3()` 统一支付执行方法

### 第5次迭代：性能优化和安全加固
- **目标**：提升性能和安全性
- **改进**：
  - 增强参数验证（appId格式、时间戳范围、package格式等）
  - 新增状态清理机制
  - 优化代码结构，减少重复代码
  - 完善组件卸载时的清理逻辑

## 核心功能特性

### 1. 参数验证
```javascript
validateWeChatPayV3Params(payData) {
    // 验证必要参数存在性
    // 验证appId格式（wx开头，18位）
    // 验证时间戳格式和范围
    // 验证随机字符串长度
    // 验证package格式
    // 验证签名类型和长度
}
```

### 2. 错误处理
```javascript
handleWeChatPayV3Error(res, data, order_id) {
    // 根据错误码返回用户友好的错误信息
    // 判断是否可重试
    // 执行重试或最终失败处理
}
```

### 3. 重试机制
- 最大重试次数：3次
- 重试延迟：递增延迟（1秒、2秒、3秒）
- 可重试错误：网络错误、系统错误等

### 4. 状态管理
- `idle`：空闲状态
- `processing`：支付处理中
- `success`：支付成功
- `failed`：支付失败

## 前后端配合要求

## 后端需要配合的升级工作

### 1. 证书和密钥配置
```php
// 需要新增的配置项
'wechat_pay_v3' => [
    'app_id' => 'wx1234567890123456',
    'mch_id' => '1234567890',
    'private_key_path' => '/path/to/apiclient_key.pem',    // 商户私钥
    'certificate_path' => '/path/to/apiclient_cert.pem',   // 商户证书
    'platform_certs_path' => '/path/to/wechatpay_cert/',   // 平台证书目录
    'api_v3_key' => 'your_api_v3_key_32_chars',           // APIv3密钥
],
```

### 2. 接口URL升级
- **v2统一下单**：`https://api.mch.weixin.qq.com/pay/unifiedorder`
- **v3 JSAPI支付**：`https://api.mch.weixin.qq.com/v3/pay/transactions/jsapi`
- **v3 APP支付**：`https://api.mch.weixin.qq.com/v3/pay/transactions/app`
- **v3 H5支付**：`https://api.mch.weixin.qq.com/v3/pay/transactions/h5`

### 3. 请求参数结构变化
**v2请求参数（XML）：**
```xml
<xml>
  <appid>wx1234567890123456</appid>
  <mch_id>1234567890</mch_id>
  <body>商品描述</body>
  <out_trade_no>20210101123456</out_trade_no>
  <total_fee>100</total_fee>
  <trade_type>JSAPI</trade_type>
  <openid>用户openid</openid>
  <sign>签名</sign>
</xml>
```

**v3请求参数（JSON）：**
```json
{
  "appid": "wx1234567890123456",
  "mchid": "1234567890",
  "description": "商品描述",
  "out_trade_no": "20210101123456",
  "amount": {
    "total": 100,
    "currency": "CNY"
  },
  "payer": {
    "openid": "用户openid"
  }
}
```

### 4. 签名算法升级
**v2签名（MD5）：**
```php
$stringA = "appid=wx123&body=test&mch_id=123&nonce_str=abc";
$stringSignTemp = $stringA . "&key=" . $api_key;
$sign = strtoupper(md5($stringSignTemp));
```

**v3签名（RSA-SHA256）：**
```php
$message = "POST
/v3/pay/transactions/jsapi
1640995200
abc123
{json_body}
";
openssl_sign($message, $signature, $private_key, OPENSSL_ALGO_SHA256);
$sign = base64_encode($signature);
```

### 5. 回调处理升级
**v2回调验证：**
```php
// 简单的签名验证
$sign = $data['sign'];
unset($data['sign']);
$mySign = $this->generateSign($data);
if ($mySign === $sign) {
    // 验证成功
}
```

**v3回调验证：**
```php
// 证书验证 + 数据解密
$headers = getallheaders();
$body = file_get_contents('php://input');

// 1. 验证证书签名
$this->verifySignature($headers, $body);

// 2. 解密回调数据
$data = json_decode($body, true);
$decrypted = $this->decryptResource($data['resource']);
```

## 前端升级工作（已完成）

### 核心修改文件

#### 1. `/components/payment/payment.vue` - 主支付组件
**修改位置和内容：**
- **第588-650行**：小程序支付处理，添加参数验证和重试机制
- **第674-740行**：H5支付处理，增强错误处理
- **methods 新增**：`validateWeChatPayV3Params()`、`handleWeChatPayV3Error()`、`executeWeChatPayV3()`等方法

#### 2. `/pages/plugins/allocation/cashier/cashier.vue` - 收银台组件
**修改位置和内容：**
- **第151-180行**：支付处理方法，添加参数验证
- **methods 新增**：参数验证和错误处理方法


## ShopXO完整升级建议

### 阶段1：后端基础升级（必须）
1. **下载微信支付证书**
   - 登录微信商户平台
   - 下载API证书（apiclient_cert.pem、apiclient_key.pem）
   - 设置APIv3密钥（32位字符）

2. **创建v3支付服务类**
   - 参考 `/docs/WECHAT_PAY_API_COMPARISON.md` 中的 `WeChatPayV3Service` 类
   - 实现JSAPI、APP、H5等支付方式
   - 处理RSA签名和证书验证

3. **修改支付插件**
   - 更新 `application/plugins/payment/weixinweb/` 目录下的文件
   - 添加v2/v3兼容性开关
   - 升级回调处理逻辑

### 阶段2：配置管理升级
1. **后台配置新增**
   - 微信支付API版本选择（v2/v3）
   - APIv3密钥配置
   - 证书路径配置

2. **数据库升级**
   ```sql
   -- 添加配置项
   INSERT INTO `__PREFIX__config` (`only_tag`, `name`, `describe`, `value`, `type`, `default_value`, `view_type`, `view_data`, `is_required`, `sort`) VALUES
   ('common', 'common_wechat_pay_use_v3', '微信支付API版本', '0', 'select', '0', 'radio', '[{\"value\":\"0\",\"name\":\"API v2\"},{\"value\":\"1\",\"name\":\"API v3\"}]', 0, 1),
   ('common', 'common_wechat_api_v3_key', 'APIv3密钥', '', 'input', '', 'text', '', 0, 2);
   ```

### 阶段3：测试验证
1. **开发环境测试**
   - 配置测试商户号
   - 验证支付流程
   - 测试回调处理

2. **多平台兼容性测试**
   - 微信小程序支付
   - H5微信内置浏览器支付
   - App原生支付

### 阶段4：生产部署
1. **灰度发布**
   - 先开启少量用户使用v3
   - 监控支付成功率
   - 确认无问题后全量切换

2. **监控告警**
   - 支付成功率监控
   - 错误日志告警
   - 回调处理异常告警

## 风险控制和应急方案

### 1. 双版本并行
- 保留v2作为备选方案
- 配置开关可快速切换
- 监控两个版本的性能差异

### 2. 回滚策略
- 数据库配置快速回滚
- 代码版本回滚方案
- 用户影响最小化

### 3. 时间规划
- **2025年3月前**：完成开发和测试
- **2025年4月**：灰度发布
- **2025年5月**：全量切换v3
- **2025年6月前**：完全停用v2

---

**详细技术对比请参考**：[微信支付API v2 vs v3详细对比](/docs/WECHAT_PAY_API_COMPARISON.md)

---

**升级完成时间**: 2025年9月5日
**升级版本**: v3.0.0
**兼容性**: 向后兼容，支持v2和v3
