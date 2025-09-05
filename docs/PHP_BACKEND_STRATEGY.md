# ShopXO PHP后端处理策略详解

## 项目背景

ShopXO原有架构中存在PHP后端系统，在重构到Next.js 15的过程中，需要制定明确的PHP后端处理策略。

## PHP后端现状分析

### 当前PHP架构
```
ShopXO PHP Backend (现有系统)
├── 用户管理系统
├── 商品管理系统  
├── 订单处理系统
├── 支付集成
├── 插件管理
├── 数据统计
└── 管理后台
```

### 业务数据流
```
UniApp前端 ↔ PHP API ↔ MySQL数据库
```

## 重构策略选择

### 方案A: 完全替换PHP后端 (推荐)

#### 优势
- **技术栈统一**: JavaScript/TypeScript全栈
- **性能优化**: Node.js + Next.js原生集成
- **开发效率**: 前后端共享代码和类型
- **现代化架构**: 微服务、容器化部署
- **生态丰富**: npm生态系统

#### 实施步骤
```typescript
// 1. 创建Next.js API Routes替换PHP接口
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '20');

  try {
    const products = await prisma.product.findMany({
      skip: (page - 1) * limit,
      take: limit,
      include: {
        category: true,
        brand: true,
        reviews: {
          take: 5,
          orderBy: { createdAt: 'desc' }
        }
      }
    });

    return NextResponse.json({
      code: 0,
      data: products,
      message: 'success'
    });
  } catch (error) {
    return NextResponse.json({
      code: 1,
      message: 'Internal server error'
    }, { status: 500 });
  }
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  
  try {
    const product = await prisma.product.create({
      data: {
        title: body.title,
        description: body.description,
        price: body.price,
        stock: body.stock,
        categoryId: body.categoryId,
        images: body.images,
        status: 'ACTIVE'
      }
    });

    return NextResponse.json({
      code: 0,
      data: product,
      message: 'Product created successfully'
    });
  } catch (error) {
    return NextResponse.json({
      code: 1,
      message: 'Failed to create product'
    }, { status: 400 });
  }
}
```

#### 核心API迁移

```typescript
// lib/api/legacy-adapter.ts
/**
 * PHP API格式适配器
 * 保持与原有UniApp前端的API契约一致
 */

interface LegacyAPIResponse<T = any> {
  code: number;
  msg: string;
  data: T;
}

export function createLegacyResponse<T>(
  data: T, 
  code: number = 0, 
  msg: string = 'success'
): LegacyAPIResponse<T> {
  return { code, msg, data };
}

// 商品列表API - 保持原有格式
export async function getProductsLegacy(params: {
  page?: number;
  limit?: number;
  category_id?: string;
  keyword?: string;
}) {
  const products = await getProducts({
    page: params.page || 1,
    limit: params.limit || 20,
    categoryId: params.category_id,
    searchTerm: params.keyword
  });

  // 转换为原有PHP API格式
  return createLegacyResponse({
    data: products.products.map(product => ({
      id: product.id,
      title: product.title,
      images: product.images,
      price: product.price.toString(),
      original_price: product.originalPrice?.toString(),
      sales_count: product.sales,
      is_favor: false, // 需要根据用户状态判断
    })),
    total: products.pagination.total,
    page_total: Math.ceil(products.pagination.total / params.limit!),
    page: params.page || 1
  });
}

// 用户登录API
export async function userLoginLegacy(params: {
  accounts: string;
  pwd: string;
  type: string;
}) {
  try {
    const result = await signIn('credentials', {
      phone: params.accounts,
      password: params.pwd,
      redirect: false
    });

    if (result?.error) {
      return createLegacyResponse(null, 1, '账号或密码错误');
    }

    // 获取用户信息
    const user = await getUserByPhone(params.accounts);
    
    return createLegacyResponse({
      user_id: user.id,
      token: result.jwt,
      user_info: {
        nickname: user.nickname,
        avatar: user.avatar,
        mobile: user.phone,
        gender: user.profile?.gender
      }
    });
  } catch (error) {
    return createLegacyResponse(null, 1, '登录失败');
  }
}
```

#### 数据迁移脚本

```typescript
// scripts/migrate-from-php.ts
import { PrismaClient } from '@prisma/client';
import mysql from 'mysql2/promise';

const prisma = new PrismaClient();

// 原有PHP数据库连接
const phpConnection = mysql.createConnection({
  host: process.env.PHP_DB_HOST,
  user: process.env.PHP_DB_USER,
  password: process.env.PHP_DB_PASSWORD,
  database: process.env.PHP_DB_NAME
});

export async function migrateUsers() {
  console.log('开始迁移用户数据...');
  
  const [rows] = await phpConnection.execute(
    'SELECT * FROM sxo_user ORDER BY id ASC'
  );

  const users = rows as any[];
  
  for (const phpUser of users) {
    try {
      await prisma.user.create({
        data: {
          id: `user_${phpUser.id}`, // 保持原有ID映射
          phone: phpUser.mobile,
          email: phpUser.email,
          username: phpUser.username,
          nickname: phpUser.nickname,
          avatar: phpUser.avatar,
          password: phpUser.pwd, // 密码需要重新加密
          status: phpUser.is_delete === 0 ? 'ACTIVE' : 'INACTIVE',
          createdAt: new Date(phpUser.add_time * 1000),
          profile: {
            create: {
              realName: phpUser.user_name_view,
              gender: phpUser.gender === 1 ? 'MALE' : phpUser.gender === 2 ? 'FEMALE' : 'OTHER',
              birthday: phpUser.birthday ? new Date(phpUser.birthday * 1000) : null,
              experience: phpUser.integral || 0,
              points: phpUser.locking_integral || 0
            }
          }
        }
      });
    } catch (error) {
      console.error(`用户 ${phpUser.id} 迁移失败:`, error);
    }
  }
  
  console.log('用户数据迁移完成');
}

export async function migrateProducts() {
  console.log('开始迁移商品数据...');
  
  const [rows] = await phpConnection.execute(`
    SELECT p.*, c.name as category_name, b.name as brand_name
    FROM sxo_goods p
    LEFT JOIN sxo_goods_category c ON p.category_id = c.id  
    LEFT JOIN sxo_goods_brand b ON p.brand_id = b.id
    WHERE p.is_delete = 0
    ORDER BY p.id ASC
  `);

  const products = rows as any[];
  
  for (const phpProduct of products) {
    try {
      // 处理图片数据
      const images = phpProduct.images ? 
        JSON.parse(phpProduct.images).map((img: any) => img.original) : 
        [];

      await prisma.product.create({
        data: {
          id: `product_${phpProduct.id}`,
          title: phpProduct.title,
          subtitle: phpProduct.simple_desc,
          description: phpProduct.desc,
          content: phpProduct.content,
          images: images,
          video: phpProduct.video,
          price: phpProduct.price,
          originalPrice: phpProduct.original_price,
          costPrice: phpProduct.cost_price,
          stock: phpProduct.inventory,
          sales: phpProduct.sales_count,
          weight: phpProduct.weight,
          status: phpProduct.is_shelves === 1 ? 'ACTIVE' : 'INACTIVE',
          categoryId: `category_${phpProduct.category_id}`,
          brandId: phpProduct.brand_id ? `brand_${phpProduct.brand_id}` : null,
          seoTitle: phpProduct.seo_title,
          seoKeywords: phpProduct.seo_keywords,
          seoDescription: phpProduct.seo_desc,
          createdAt: new Date(phpProduct.add_time * 1000),
          publishedAt: phpProduct.is_shelves === 1 ? new Date(phpProduct.upd_time * 1000) : null
        }
      });
    } catch (error) {
      console.error(`商品 ${phpProduct.id} 迁移失败:`, error);
    }
  }
  
  console.log('商品数据迁移完成');
}

// 运行迁移
async function runMigration() {
  try {
    await migrateUsers();
    await migrateProducts();
    // 继续迁移其他数据...
  } catch (error) {
    console.error('数据迁移失败:', error);
  } finally {
    await prisma.$disconnect();
    await phpConnection.end();
  }
}

runMigration();
```

### 方案B: 渐进式迁移 (备选)

#### 实施策略
```
阶段1: 新功能用Next.js API，老功能保持PHP
阶段2: 逐步迁移核心业务到Next.js
阶段3: 完全替换PHP后端
```

#### API代理实现
```typescript
// lib/api/php-proxy.ts
import { NextRequest } from 'next/server';

const PHP_API_BASE = process.env.PHP_API_URL || 'https://api.shopxo.com';

export async function proxyToPHP(
  request: NextRequest,
  endpoint: string
) {
  const url = new URL(request.url);
  const phpUrl = `${PHP_API_BASE}${endpoint}${url.search}`;

  try {
    const response = await fetch(phpUrl, {
      method: request.method,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': request.headers.get('Authorization') || '',
      },
      body: request.method !== 'GET' ? await request.text() : undefined,
    });

    return response;
  } catch (error) {
    console.error('PHP API代理失败:', error);
    throw new Error('Service temporarily unavailable');
  }
}

// app/api/legacy/[...slug]/route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: { slug: string[] } }
) {
  const endpoint = '/' + params.slug.join('/');
  return proxyToPHP(request, endpoint);
}

export async function POST(
  request: NextRequest,
  { params }: { params: { slug: string[] } }
) {
  const endpoint = '/' + params.slug.join('/');
  return proxyToPHP(request, endpoint);
}
```

## 核心系统迁移详解

### 1. 用户认证系统

```typescript
// lib/auth/legacy-auth.ts
import bcrypt from 'bcryptjs';
import { NextAuthConfig } from 'next-auth';

export const legacyAuthConfig: NextAuthConfig = {
  providers: [
    {
      id: 'shopxo-legacy',
      name: 'ShopXO Legacy',
      type: 'credentials',
      credentials: {
        accounts: { label: "账号", type: "text" },
        pwd: { label: "密码", type: "password" },
        type: { label: "类型", type: "text" }
      },
      async authorize(credentials) {
        // 兼容原有登录逻辑
        const user = await prisma.user.findFirst({
          where: {
            OR: [
              { phone: credentials.accounts as string },
              { email: credentials.accounts as string },
              { username: credentials.accounts as string }
            ]
          },
          include: { profile: true }
        });

        if (user && await bcrypt.compare(credentials.pwd as string, user.password || '')) {
          return {
            id: user.id,
            name: user.nickname || user.username,
            email: user.email,
            image: user.avatar,
          };
        }
        return null;
      }
    }
  ]
};
```

### 2. 订单处理系统

```typescript
// lib/api/orders.ts
export async function createOrderLegacy(orderData: {
  user_id: string;
  goods_list: Array<{
    goods_id: string;
    spec: any[];
    stock: number;
  }>;
  address_id: string;
  payment_id: string;
  voucher_id?: string;
}) {
  try {
    // 1. 验证商品库存
    for (const item of orderData.goods_list) {
      const product = await prisma.product.findUnique({
        where: { id: item.goods_id },
        include: { skus: true }
      });

      if (!product || product.stock < item.stock) {
        return createLegacyResponse(null, 1, '商品库存不足');
      }
    }

    // 2. 创建订单
    const order = await prisma.order.create({
      data: {
        orderNo: generateOrderNo(),
        userId: orderData.user_id,
        status: 'PENDING_PAYMENT',
        paymentStatus: 'PENDING',
        totalAmount: calculateTotal(orderData.goods_list),
        actualAmount: calculateActual(orderData.goods_list, orderData.voucher_id),
        // ... 其他字段
        items: {
          create: orderData.goods_list.map(item => ({
            productId: item.goods_id,
            quantity: item.stock,
            price: getProductPrice(item.goods_id),
            // ... 其他字段
          }))
        }
      },
      include: { items: true }
    });

    // 3. 更新库存
    await updateProductStock(orderData.goods_list);

    return createLegacyResponse({
      order_id: order.id,
      order_no: order.orderNo,
      pay_url: generatePaymentUrl(order.id)
    });
  } catch (error) {
    return createLegacyResponse(null, 1, '订单创建失败');
  }
}
```

### 3. 支付系统集成

```typescript
// lib/payment/adapters.ts
export class PaymentAdapter {
  static async processPayment(params: {
    order_id: string;
    payment_type: 'wechat' | 'alipay' | 'balance';
    amount: number;
  }) {
    const order = await prisma.order.findUnique({
      where: { id: params.order_id }
    });

    if (!order) {
      throw new Error('订单不存在');
    }

    switch (params.payment_type) {
      case 'wechat':
        return await this.processWechatPay(order, params.amount);
      case 'alipay':
        return await this.processAlipay(order, params.amount);
      case 'balance':
        return await this.processBalancePay(order, params.amount);
      default:
        throw new Error('不支持的支付方式');
    }
  }

  private static async processWechatPay(order: Order, amount: number) {
    // 微信支付逻辑
    const wechatPay = new WechatPayService();
    const result = await wechatPay.createOrder({
      out_trade_no: order.orderNo,
      total_fee: Math.round(amount * 100), // 转为分
      body: `ShopXO订单-${order.orderNo}`,
      notify_url: `${process.env.APP_URL}/api/payment/wechat/notify`
    });

    return {
      payment_id: result.prepay_id,
      payment_params: result.payment_params
    };
  }
}
```

### 4. 插件系统迁移

```typescript
// lib/plugins/legacy-loader.ts
export class LegacyPluginLoader {
  static async loadPluginConfig(pluginName: string) {
    // 从数据库加载原有插件配置
    const config = await prisma.plugin.findUnique({
      where: { name: pluginName }
    });

    if (!config) return null;

    // 转换为新的插件格式
    return {
      id: pluginName,
      name: config.name,
      version: config.version,
      config: config.config,
      enabled: config.isEnabled,
      // 映射原有配置到新的插件系统
      routes: this.mapLegacyRoutes(config.config),
      components: this.mapLegacyComponents(config.config)
    };
  }

  private static mapLegacyRoutes(config: any) {
    // 将PHP插件路由映射到Next.js路由
    return config.routes?.map((route: any) => ({
      path: `/plugins/${route.path}`,
      component: route.component,
      meta: route.meta
    })) || [];
  }
}
```

## 数据库兼容性处理

### 字段映射策略

```typescript
// lib/db/field-mapping.ts
export const FIELD_MAPPINGS = {
  // 用户表字段映射
  user: {
    'add_time': 'createdAt',
    'upd_time': 'updatedAt', 
    'is_delete': 'status',
    'mobile': 'phone',
    'pwd': 'password',
    'nickname': 'nickname'
  },
  
  // 商品表字段映射
  goods: {
    'simple_desc': 'subtitle',
    'desc': 'description',
    'original_price': 'originalPrice',
    'cost_price': 'costPrice',
    'inventory': 'stock',
    'sales_count': 'sales',
    'is_shelves': 'status'
  }
};

export function mapLegacyFields(tableName: string, data: any) {
  const mapping = FIELD_MAPPINGS[tableName];
  if (!mapping) return data;

  const mapped = {};
  for (const [legacyField, newField] of Object.entries(mapping)) {
    if (data[legacyField] !== undefined) {
      mapped[newField] = data[legacyField];
    }
  }
  
  return { ...data, ...mapped };
}
```

### 数据同步服务

```typescript
// lib/sync/data-sync.ts
export class DataSyncService {
  static async syncFromPHP() {
    console.log('开始数据同步...');
    
    try {
      // 同步用户数据
      await this.syncUsers();
      
      // 同步商品数据
      await this.syncProducts();
      
      // 同步订单数据
      await this.syncOrders();
      
      console.log('数据同步完成');
    } catch (error) {
      console.error('数据同步失败:', error);
      throw error;
    }
  }

  private static async syncUsers() {
    const phpUsers = await this.fetchFromPHP('/api/users/sync');
    
    for (const phpUser of phpUsers) {
      await prisma.user.upsert({
        where: { id: `user_${phpUser.id}` },
        update: mapLegacyFields('user', phpUser),
        create: {
          id: `user_${phpUser.id}`,
          ...mapLegacyFields('user', phpUser)
        }
      });
    }
  }

  private static async fetchFromPHP(endpoint: string) {
    const response = await fetch(`${process.env.PHP_API_URL}${endpoint}`, {
      headers: {
        'Authorization': `Bearer ${process.env.PHP_API_TOKEN}`
      }
    });
    
    if (!response.ok) {
      throw new Error(`PHP API请求失败: ${response.statusText}`);
    }
    
    return response.json();
  }
}
```

## 性能优化策略

### 缓存PHP数据

```typescript
// lib/cache/php-cache.ts
export class PHPCacheAdapter {
  static async getOrSetFromPHP<T>(
    key: string,
    phpEndpoint: string,
    ttl: number = 3600
  ): Promise<T> {
    // 先从Redis获取
    const cached = await redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    // 从PHP API获取
    const data = await fetch(`${process.env.PHP_API_URL}${phpEndpoint}`);
    const result = await data.json();

    // 存入Redis
    await redis.setex(key, ttl, JSON.stringify(result));
    
    return result;
  }
}
```

## 部署和运维

### Docker配置

```dockerfile
# 渐进式迁移期间的Docker配置
version: '3.8'
services:
  nextjs:
    build: .
    ports:
      - "3000:3000"
    environment:
      - PHP_API_URL=http://php-backend:80
      - DATABASE_URL=postgresql://user:pass@postgres:5432/shopxo
    depends_on:
      - postgres
      - redis
      - php-backend

  php-backend:
    image: shopxo/php:latest
    ports:
      - "8080:80"
    volumes:
      - ./php-app:/var/www/html
    depends_on:
      - mysql

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: shopxo
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: shopxo_legacy
      MYSQL_USER: user
      MYSQL_PASSWORD: pass

  redis:
    image: redis:7-alpine
```

## 总结建议

### 推荐实施路径

1. **完全替换策略** (推荐)
   - 技术债务最少
   - 长期维护成本最低
   - 性能和开发效率最优

2. **迁移时间安排**
   - 数据迁移: 2周
   - API重写: 4-6周  
   - 测试验证: 2周
   - 上线部署: 1周

3. **风险控制**
   - 完整的数据备份
   - 分阶段上线验证
   - 回滚机制准备
   - 监控告警系统

通过这套完整的PHP后端处理策略，可以确保从旧系统到新系统的平滑过渡，同时获得现代化技术栈带来的所有优势。