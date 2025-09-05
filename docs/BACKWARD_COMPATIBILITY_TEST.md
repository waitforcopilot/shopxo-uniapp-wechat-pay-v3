# 微信支付 API v2/v3 向后兼容性测试指南

## 测试目标
验证升级到微信支付 API v3 后，现有的 v2 格式参数仍能正常工作，确保零停机升级。

## 测试场景

### 1. 主支付组件 (payment.vue) 兼容性测试

#### 场景1：v2 格式参数测试
```javascript
// 模拟v2格式参数（MD5签名）
const v2Params = {
    timeStamp: "1640995200",
    nonceStr: "abc123def456",
    package: "prepay_id=wx123456789",
    signType: "MD5",
    paySign: "A1B2C3D4E5F6G7H8"
};

// 预期结果：正常支付，控制台显示"检测到微信支付v2格式参数"
```

#### 场景2：v3 格式参数测试
```javascript
// 模拟v3格式参数（RSA签名）
const v3Params = {
    appId: "wx1234567890abcdef",
    timeStamp: "1640995200",
    nonceStr: "xyz789uvw012",
    package: "prepay_id=wx987654321",
    signType: "RSA",
    paySign: "RSA_SIGNATURE_STRING_HERE"
};

// 预期结果：正常支付，控制台显示"检测到微信支付v3格式参数"
```

#### 场景3：无效参数测试
```javascript
// 缺少必要参数
const invalidParams = {
    timeStamp: "1640995200"
    // 缺少其他必要参数
};

// 预期结果：支付失败，显示"微信支付参数验证失败"
```

### 2. 收银台组件 (cashier.vue) 兼容性测试

#### 场景1：收银台v2兼容性
```javascript
// 测试收银台组件处理v2格式参数
this.data.pay_data = {
    timeStamp: "1640995200",
    nonceStr: "cashier123",
    package: "prepay_id=wx_cashier_001",
    signType: "MD5",
    paySign: "CASHIER_MD5_SIGN"
};
```

#### 场景2：收银台v3兼容性
```javascript
// 测试收银台组件处理v3格式参数
this.data.pay_data = {
    appId: "wx1234567890abcdef",
    timeStamp: "1640995200",
    nonceStr: "cashier_v3_456",
    package: "prepay_id=wx_cashier_v3_002",
    signType: "RSA",
    paySign: "CASHIER_RSA_SIGNATURE"
};
```

### 3. H5 平台兼容性测试

#### 场景1：微信浏览器环境测试
```javascript
// 在微信浏览器中测试支付
// 验证 WeixinJSBridge.invoke 调用是否正常
// 检查参数格式自动适配是否生效
```

#### 场景2：外部浏览器环境测试
```javascript
// 在非微信浏览器中测试
// 验证跳转到微信支付页面是否正常
```

## 测试步骤

### 步骤1：环境准备
1. 确保项目已集成微信支付 v3 升级代码
2. 配置测试用的微信商户号和证书
3. 准备 v2 和 v3 格式的测试参数

### 步骤2：单元测试
```javascript
// 在浏览器控制台或测试环境中执行
function testParameterValidation() {
    // 导入支付组件
    const paymentComponent = require('@/components/payment/payment.vue');
    
    // 测试v2参数验证
    const v2Valid = paymentComponent.methods.validateAndNormalizeWeChatPayParams(v2TestParams);
    console.log('v2参数验证结果:', v2Valid);
    
    // 测试v3参数验证
    const v3Valid = paymentComponent.methods.validateAndNormalizeWeChatPayParams(v3TestParams);
    console.log('v3参数验证结果:', v3Valid);
}
```

### 步骤3：集成测试
1. 在开发环境中启动应用
2. 进入商品详情页，点击立即购买
3. 选择微信支付，观察支付参数格式检测
4. 重复测试收银台页面的支付功能

### 步骤4：真实支付测试
1. 使用微信支付沙箱环境进行测试
2. 分别测试 v2 和 v3 格式参数的支付流程
3. 验证支付成功后的回调处理

## 预期结果

### 成功标准
- ✅ v2 格式参数能够正常发起支付
- ✅ v3 格式参数能够正常发起支付
- ✅ 参数格式自动检测工作正常
- ✅ 错误处理保持一致的用户体验
- ✅ 控制台日志正确标识参数格式

### 失败处理
- ❌ 如果v2参数无法支付，检查参数适配逻辑
- ❌ 如果v3参数报错，验证RSA签名处理
- ❌ 如果格式检测错误，调试检测算法

## 回滚方案
如果兼容性测试失败，可以通过以下方式快速回滚：

1. **代码回滚**：恢复到升级前的版本
2. **分支切换**：使用Git切换到稳定分支
3. **配置回滚**：临时禁用v3相关功能

## 监控指标
- 支付成功率对比（升级前后）
- 支付失败原因分析
- 用户支付体验反馈
- 系统错误日志数量

## 注意事项
1. 测试过程中不要使用真实的生产环境密钥
2. 确保测试数据不会影响生产订单
3. 记录所有测试结果，便于问题追踪
4. 在生产环境部署前，必须完成所有测试场景

## 技术支持
如遇到兼容性问题，请检查：
1. 微信支付商户号配置是否正确
2. API密钥和证书是否有效
3. 签名算法是否匹配参数格式
4. UniApp版本是否支持所用的支付API

---
*此文档应在每次微信支付相关更新后进行复查和更新*
