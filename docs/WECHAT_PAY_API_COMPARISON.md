# 微信支付 API v2 vs v3 详细对比分析

## 概述对比

| 维度 | API v2 | API v3 |
|------|--------|--------|
| 发布时间 | 2014年 | 2020年 |
| 安全性 | MD5/HMAC-SHA256签名 | RSA/ECDSA数字签名 |
| 接口风格 | XML格式 | RESTful + JSON |
| 证书要求 | 可选 | 强制使用 |
| 回调验证 | 签名验证 | 证书验证 |
| 支持状态 | 2025年6月30日停止服务 | 当前主推版本 |

## 1. 技术架构差异

### 1.1 数据格式
**API v2 (XML):**
```xml
<xml>
  <appid><![CDATA[wx1234567890123456]]></appid>
  <mch_id><![CDATA[1234567890]]></mch_id>
  <nonce_str><![CDATA[abc123]]></nonce_str>
  <sign><![CDATA[C380BEC2BFD727A4B6845133519F3AD6]]></sign>
  <body><![CDATA[测试商品]]></body>
  <out_trade_no><![CDATA[20210101123456]]></out_trade_no>
  <total_fee>100</total_fee>
</xml>
```

**API v3 (JSON):**
```json
{
  "appid": "wx1234567890123456",
  "mchid": "1234567890",
  "description": "测试商品",
  "out_trade_no": "20210101123456",
  "amount": {
    "total": 100,
    "currency": "CNY"
  }
}
```

### 1.2 签名算法差异

**API v2 签名算法:**
```javascript
// 1. 参数按key排序
// 2. 拼接成字符串
const stringA = "appid=wx123&body=test&mch_id=123&nonce_str=abc";
// 3. 加上密钥
const stringSignTemp = stringA + "&key=" + API_KEY;
// 4. MD5加密
const sign = MD5(stringSignTemp).toUpperCase();
```

**API v3 签名算法:**
```javascript
// 1. 构造签名串
const method = "POST";
const url = "/v3/pay/transactions/jsapi";
const timestamp = Math.floor(Date.now() / 1000);
const nonce_str = generateNonceStr();
const body = JSON.stringify(requestData);

const message = `${method}\n${url}\n${timestamp}\n${nonce_str}\n${body}\n`;

// 2. 使用商户私钥进行RSA-SHA256签名
const signature = crypto.sign("sha256", Buffer.from(message), privateKey);
```

## 2. 接口URL差异

### 2.1 统一下单接口

**API v2:**
```
POST https://api.mch.weixin.qq.com/pay/unifiedorder
```

**API v3:**
```
POST https://api.mch.weixin.qq.com/v3/pay/transactions/jsapi    # JSAPI支付
POST https://api.mch.weixin.qq.com/v3/pay/transactions/app     # APP支付
POST https://api.mch.weixin.qq.com/v3/pay/transactions/h5      # H5支付
POST https://api.mch.weixin.qq.com/v3/pay/transactions/native  # Native支付
```

### 2.2 查询订单接口

**API v2:**
```
POST https://api.mch.weixin.qq.com/pay/orderquery
```

**API v3:**
```
GET https://api.mch.weixin.qq.com/v3/pay/transactions/out-trade-no/{out_trade_no}
GET https://api.mch.weixin.qq.com/v3/pay/transactions/id/{transaction_id}
```

## 3. 请求参数差异

### 3.1 JSAPI支付参数对比

**API v2 请求参数:**
```xml
<xml>
  <appid>微信公众号或小程序的appid</appid>
  <mch_id>商户号</mch_id>
  <nonce_str>随机字符串</nonce_str>
  <sign>签名</sign>
  <body>商品描述</body>
  <out_trade_no>商户订单号</out_trade_no>
  <total_fee>订单金额(分)</total_fee>
  <spbill_create_ip>用户端IP</spbill_create_ip>
  <notify_url>回调地址</notify_url>
  <trade_type>JSAPI</trade_type>
  <openid>用户openid</openid>
</xml>
```

**API v3 请求参数:**
```json
{
  "appid": "wx1234567890123456",
  "mchid": "1234567890",
  "description": "商品描述",
  "out_trade_no": "商户订单号",
  "notify_url": "回调地址",
  "amount": {
    "total": 100,
    "currency": "CNY"
  },
  "payer": {
    "openid": "用户openid"
  }
}
```

### 3.2 返回参数对比

**API v2 返回:**
```xml
<xml>
  <return_code><![CDATA[SUCCESS]]></return_code>
  <return_msg><![CDATA[OK]]></return_msg>
  <appid><![CDATA[wx1234567890123456]]></appid>
  <mch_id><![CDATA[1234567890]]></mch_id>
  <result_code><![CDATA[SUCCESS]]></result_code>
  <prepay_id><![CDATA[wx123456789012345678901234567890]]></prepay_id>
  <trade_type><![CDATA[JSAPI]]></trade_type>
</xml>
```

**API v3 返回:**
```json
{
  "prepay_id": "wx123456789012345678901234567890"
}
```

## 4. 前端调用参数差异

### 4.1 小程序端调用参数

**API v2:**
```javascript
uni.requestPayment({
  timeStamp: timestamp,
  nonceStr: nonceStr,
  package: `prepay_id=${prepayId}`,
  signType: 'MD5',  // 或 HMAC-SHA256
  paySign: paySign
});
```

**API v3:**
```javascript
uni.requestPayment({
  timeStamp: timestamp,
  nonceStr: nonceStr,
  package: `prepay_id=${prepayId}`,
  signType: 'RSA',  // 必须是RSA
  paySign: paySign
});
```

### 4.2 H5端调用参数

**API v2:**
```javascript
WeixinJSBridge.invoke('getBrandWCPayRequest', {
  appId: appId,
  timeStamp: timestamp,
  nonceStr: nonceStr,
  package: `prepay_id=${prepayId}`,
  signType: 'MD5',
  paySign: paySign
});
```

**API v3:**
```javascript
WeixinJSBridge.invoke('getBrandWCPayRequest', {
  appId: appId,
  timeStamp: timestamp,
  nonceStr: nonceStr,
  package: `prepay_id=${prepayId}`,
  signType: 'RSA',
  paySign: paySign
});
```

## 5. 回调通知差异

### 5.1 回调数据格式

**API v2 回调 (XML):**
```xml
<xml>
  <appid><![CDATA[wx1234567890123456]]></appid>
  <bank_type><![CDATA[CFT]]></bank_type>
  <cash_fee><![CDATA[100]]></cash_fee>
  <fee_type><![CDATA[CNY]]></fee_type>
  <is_subscribe><![CDATA[N]]></is_subscribe>
  <mch_id><![CDATA[1234567890]]></mch_id>
  <nonce_str><![CDATA[abc123]]></nonce_str>
  <openid><![CDATA[oUpF8uMuAJO_M2pxb1Q9zNjWeS6o]]></openid>
  <out_trade_no><![CDATA[20210101123456]]></out_trade_no>
  <result_code><![CDATA[SUCCESS]]></result_code>
  <return_code><![CDATA[SUCCESS]]></return_code>
  <sign><![CDATA[B552ED6B0B3BC37D30DDCB19526A2B]]></sign>
  <time_end><![CDATA[20210101120000]]></time_end>
  <total_fee>100</total_fee>
  <trade_type><![CDATA[JSAPI]]></trade_type>
  <transaction_id><![CDATA[4200001234567890]]></transaction_id>
</xml>
```

**API v3 回调 (JSON):**
```json
{
  "id": "EV-2018022511223320873",
  "create_time": "2021-01-01T12:00:00+08:00",
  "resource_type": "encrypt-resource",
  "event_type": "TRANSACTION.SUCCESS",
  "summary": "支付成功",
  "resource": {
    "original_type": "transaction",
    "algorithm": "AEAD_AES_256_GCM",
    "ciphertext": "...",
    "associated_data": "transaction",
    "nonce": "..."
  }
}
```

### 5.2 回调验证方式

**API v2:**
```php
// 验证签名
$sign = $data['sign'];
unset($data['sign']);
ksort($data);
$stringA = urldecode(http_build_query($data));
$stringSignTemp = $stringA . "&key=" . $key;
$mySign = strtoupper(md5($stringSignTemp));
if ($mySign === $sign) {
    // 验证成功
}
```

**API v3:**
```php
// 验证证书和签名
$timestamp = $_SERVER['HTTP_WECHATPAY_TIMESTAMP'];
$nonce = $_SERVER['HTTP_WECHATPAY_NONCE'];
$signature = $_SERVER['HTTP_WECHATPAY_SIGNATURE'];
$serial = $_SERVER['HTTP_WECHATPAY_SERIAL'];
$body = file_get_contents('php://input');

$message = $timestamp . "\n" . $nonce . "\n" . $body . "\n";
$verify_result = openssl_verify(
    $message,
    base64_decode($signature),
    $publicKey,
    OPENSSL_ALGO_SHA256
);
```

## 6. ShopXO升级API v3详细方案

### 6.1 后端升级步骤

#### 步骤1：证书配置
```php
// config/payment.php 添加v3配置
'wechat_v3' => [
    'app_id' => 'wx1234567890123456',
    'mch_id' => '1234567890',
    'private_key_path' => '/path/to/apiclient_key.pem',    // 商户私钥
    'certificate_path' => '/path/to/apiclient_cert.pem',   // 商户证书
    'platform_certs_path' => '/path/to/wechatpay_cert/',   // 平台证书目录
    'api_v3_key' => 'your_api_v3_key_32_chars',           // APIv3密钥
],
```

#### 步骤2：创建v3支付类
```php
// application/service/WeChatPayV3Service.php
class WeChatPayV3Service
{
    private $appId;
    private $mchId;
    private $privateKey;
    private $apiV3Key;
    
    public function __construct($config)
    {
        $this->appId = $config['app_id'];
        $this->mchId = $config['mch_id'];
        $this->privateKey = file_get_contents($config['private_key_path']);
        $this->apiV3Key = $config['api_v3_key'];
    }
    
    /**
     * JSAPI支付下单
     */
    public function jsapiPay($orderData)
    {
        $url = 'https://api.mch.weixin.qq.com/v3/pay/transactions/jsapi';
        $data = [
            'appid' => $this->appId,
            'mchid' => $this->mchId,
            'description' => $orderData['description'],
            'out_trade_no' => $orderData['out_trade_no'],
            'notify_url' => $orderData['notify_url'],
            'amount' => [
                'total' => $orderData['total_fee'],
                'currency' => 'CNY'
            ],
            'payer' => [
                'openid' => $orderData['openid']
            ]
        ];
        
        $response = $this->request('POST', $url, $data);
        return $response;
    }
    
    /**
     * 生成前端调用参数
     */
    public function getJsApiParameters($prepayId)
    {
        $timeStamp = time();
        $nonceStr = $this->generateNonceStr();
        $package = 'prepay_id=' . $prepayId;
        
        $message = $this->appId . "\n" . $timeStamp . "\n" . $nonceStr . "\n" . $package . "\n";
        $paySign = $this->sign($message);
        
        return [
            'appId' => $this->appId,
            'timeStamp' => (string)$timeStamp,
            'nonceStr' => $nonceStr,
            'package' => $package,
            'signType' => 'RSA',
            'paySign' => $paySign
        ];
    }
    
    /**
     * RSA签名
     */
    private function sign($message)
    {
        openssl_sign($message, $signature, $this->privateKey, OPENSSL_ALGO_SHA256);
        return base64_encode($signature);
    }
    
    /**
     * 发送HTTP请求
     */
    private function request($method, $url, $data = [])
    {
        $timestamp = time();
        $nonceStr = $this->generateNonceStr();
        $body = empty($data) ? '' : json_encode($data, JSON_UNESCAPED_UNICODE);
        
        // 构造签名串
        $urlParts = parse_url($url);
        $canonical_url = $urlParts['path'];
        if (isset($urlParts['query'])) {
            $canonical_url .= '?' . $urlParts['query'];
        }
        
        $message = $method . "\n" . $canonical_url . "\n" . $timestamp . "\n" . $nonceStr . "\n" . $body . "\n";
        $signature = $this->sign($message);
        
        // 构造Authorization头
        $authorization = sprintf(
            'WECHATPAY2-SHA256-RSA2048 mchid="%s",nonce_str="%s",signature="%s",timestamp="%d",serial_no="%s"',
            $this->mchId,
            $nonceStr,
            $signature,
            $timestamp,
            $this->getSerialNo()
        );
        
        // 发送请求
        $headers = [
            'Content-Type: application/json',
            'Accept: application/json',
            'Authorization: ' . $authorization,
            'User-Agent: ShopXO-WeChatPay-V3'
        ];
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        
        if (!empty($body)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, $body);
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode !== 200) {
            throw new Exception('HTTP Error: ' . $httpCode . ', Response: ' . $response);
        }
        
        return json_decode($response, true);
    }
}
```

#### 步骤3：修改支付控制器
```php
// application/plugins/payment/weixinweb/WeixinwebService.php
class WeixinwebService
{
    public function Pay($params = [])
    {
        // 判断使用v2还是v3
        $useV3 = MyC('common_wechat_pay_use_v3', 0);
        
        if ($useV3 == 1) {
            return $this->payV3($params);
        } else {
            return $this->payV2($params);
        }
    }
    
    private function payV3($params)
    {
        $config = [
            'app_id' => MyC('common_wechat_appid'),
            'mch_id' => MyC('common_wechat_mch_id'),
            'private_key_path' => ROOT_PATH . MyC('common_wechat_apiclient_key_path'),
            'certificate_path' => ROOT_PATH . MyC('common_wechat_apiclient_cert_path'),
            'api_v3_key' => MyC('common_wechat_api_v3_key'),
        ];
        
        $wechatPay = new WeChatPayV3Service($config);
        
        $orderData = [
            'description' => $params['name'],
            'out_trade_no' => $params['order_no'],
            'total_fee' => intval($params['total_price'] * 100),
            'notify_url' => $params['call_back_url'],
            'openid' => $this->GetUserOpenid($params)
        ];
        
        try {
            $result = $wechatPay->jsapiPay($orderData);
            
            if (isset($result['prepay_id'])) {
                $jsApiParams = $wechatPay->getJsApiParameters($result['prepay_id']);
                
                return DataReturn('success', 0, [
                    'data' => $jsApiParams,
                    'msg' => '支付参数获取成功',
                    'order_no' => $params['order_no']
                ]);
            } else {
                return DataReturn('获取预支付ID失败', -1);
            }
        } catch (Exception $e) {
            return DataReturn('支付接口调用失败：' . $e->getMessage(), -1);
        }
    }
}
```

#### 步骤4：回调处理升级
```php
// application/plugins/payment/weixinweb/WeixinwebService.php
public function Respond($params = [])
{
    $useV3 = MyC('common_wechat_pay_use_v3', 0);
    
    if ($useV3 == 1) {
        return $this->respondV3($params);
    } else {
        return $this->respondV2($params);
    }
}

private function respondV3($params)
{
    try {
        // 获取回调数据
        $headers = getallheaders();
        $body = file_get_contents('php://input');
        
        // 验证签名
        if (!$this->verifyV3Signature($headers, $body)) {
            return false;
        }
        
        // 解密回调数据
        $data = json_decode($body, true);
        $resource = $data['resource'];
        
        $decryptedData = $this->decryptV3Resource(
            $resource['ciphertext'],
            $resource['nonce'],
            $resource['associated_data']
        );
        
        $orderData = json_decode($decryptedData, true);
        
        // 处理支付成功逻辑
        if ($orderData['trade_state'] === 'SUCCESS') {
            $orderNo = $orderData['out_trade_no'];
            $transactionId = $orderData['transaction_id'];
            
            // 更新订单状态
            $this->updateOrderStatus($orderNo, $transactionId);
        }
        
        // 返回成功响应
        return [
            'code' => 'SUCCESS',
            'message' => '成功'
        ];
        
    } catch (Exception $e) {
        return [
            'code' => 'FAIL',
            'message' => $e->getMessage()
        ];
    }
}

private function verifyV3Signature($headers, $body)
{
    $timestamp = $headers['Wechatpay-Timestamp'];
    $nonce = $headers['Wechatpay-Nonce'];
    $signature = $headers['Wechatpay-Signature'];
    $serial = $headers['Wechatpay-Serial'];
    
    $message = $timestamp . "\n" . $nonce . "\n" . $body . "\n";
    
    // 获取微信支付平台证书公钥
    $publicKey = $this->getPlatformPublicKey($serial);
    
    $verify_result = openssl_verify(
        $message,
        base64_decode($signature),
        $publicKey,
        OPENSSL_ALGO_SHA256
    );
    
    return $verify_result === 1;
}

private function decryptV3Resource($ciphertext, $nonce, $associated_data)
{
    $apiV3Key = MyC('common_wechat_api_v3_key');
    
    $decrypted = openssl_decrypt(
        base64_decode($ciphertext),
        'aes-256-gcm',
        $apiV3Key,
        OPENSSL_RAW_DATA,
        $nonce,
        $associated_data
    );
    
    return $decrypted;
}
```

### 6.2 前端已完成的升级

前端已通过5次迭代完成升级，主要改进：

1. **参数验证增强**：验证appId格式、时间戳范围、package格式等
2. **错误处理优化**：针对v3特定错误码的处理
3. **重试机制**：网络错误自动重试，最多3次
4. **状态管理**：idle、processing、success、failed状态管理
5. **安全加固**：参数格式验证、签名验证等

### 6.3 配置管理升级

#### 后台配置项新增：
```php
// 系统配置新增项
[
    'common_wechat_pay_use_v3' => [
        'only_tag' => 'common',
        'name' => '微信支付API版本',
        'describe' => '选择使用微信支付API v2或v3',
        'type' => 'select',
        'default_value' => '0',
        'view_type' => 'radio',
        'view_data' => [
            0 => 'API v2',
            1 => 'API v3'
        ],
        'view_data_default' => 0,
    ],
    'common_wechat_api_v3_key' => [
        'only_tag' => 'common',
        'name' => 'APIv3密钥',
        'describe' => '微信支付APIv3密钥，32位字符',
        'type' => 'input',
        'default_value' => '',
        'is_required' => 0,
    ],
    'common_wechat_apiclient_key_path' => [
        'only_tag' => 'common',
        'name' => '商户私钥路径',
        'describe' => 'apiclient_key.pem文件路径',
        'type' => 'input',
        'default_value' => '',
        'is_required' => 0,
    ],
    'common_wechat_apiclient_cert_path' => [
        'only_tag' => 'common',
        'name' => '商户证书路径',
        'describe' => 'apiclient_cert.pem文件路径',
        'type' => 'input',
        'default_value' => '',
        'is_required' => 0,
    ]
]
```

## 7. 升级检查清单

### 7.1 后端检查项
- [ ] 下载并配置商户API证书
- [ ] 设置APIv3密钥
- [ ] 创建WeChatPayV3Service类
- [ ] 修改支付控制器支持v3
- [ ] 升级回调处理逻辑
- [ ] 更新配置管理项
- [ ] 测试支付流程

### 7.2 前端检查项
- [x] 更新支付参数验证
- [x] 优化错误处理机制
- [x] 添加重试机制
- [x] 完善状态管理
- [x] 增强安全验证

### 7.3 部署检查项
- [ ] 备份现有支付配置
- [ ] 更新生产环境证书
- [ ] 配置回调URL
- [ ] 测试各平台支付
- [ ] 监控支付成功率

## 8. 风险控制

### 8.1 平滑升级策略
1. **配置开关**：保留v2和v3双重支持
2. **灰度发布**：先在测试环境验证
3. **监控告警**：设置支付成功率监控
4. **快速回滚**：保留v2作为备选方案

### 8.2 兼容性保障
1. **向后兼容**：保持原有接口不变
2. **参数映射**：自动转换v2/v3参数格式
3. **错误处理**：统一错误码映射
4. **日志记录**：详细记录升级过程

---

**升级时间规划**：建议在2025年5月前完成升级，以应对微信支付v2的停服时间（2025年6月30日）。
