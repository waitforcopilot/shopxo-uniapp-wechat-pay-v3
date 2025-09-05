# ShopXO UniApp 到 Next.js 15 完整重构技术方案

## 项目概述

**原项目**: ShopXO UniApp v6.6.0 - 跨平台电商移动应用
**目标架构**: Next.js 15 + React 18 + NextAuth.js + TypeScript
**重构目标**: 现代化架构升级，保持业务完整性，大幅提升性能

---

## 第一部分：项目架构分析

### 1.1 原有UniApp架构分析

#### 核心特点
- **全局配置驱动**: App.vue集中管理主题、API端点、语言等配置
- **插件模块化**: 30+插件系统，每个插件自包含页面和组件
- **DIY可视化系统**: 40+拖拽组件，支持动态页面构建
- **多平台兼容**: 支持小程序、H5、原生App
- **主题系统**: 8种主题色彩，动态切换
- **国际化**: Vue I18n实现中英文切换

#### 关键配置文件
- `App.vue` - 全局配置、API端点、主题设置
- `manifest.json` - 平台特定配置和应用元数据
- `pages.json` - 页面路由和tabbar配置
- `vue.config.js` - Webpack性能优化
- `locale/index.js` - 多语言配置

#### 目录结构分析
```
shopxo-uniapp-v6.6.0/
├── pages/                    # 所有应用页面
│   ├── index/               # 首页
│   ├── goods-category/      # 商品分类
│   ├── cart/                # 购物车
│   ├── user/                # 用户中心
│   ├── plugins/             # 插件页面(30+插件)
│   │   ├── seckill/        # 秒杀
│   │   ├── coupon/         # 优惠券
│   │   ├── distribution/   # 分销
│   │   ├── wallet/         # 钱包
│   │   └── ...
│   └── diy/                # DIY可视化组件
├── components/              # 60+全局组件
├── common/                  # 共享JS工具和全局样式
├── static/images/           # 主题图片资源
├── uni_modules/             # UniApp生态模块
└── locale/                  # 多语言文件
```

### 1.2 业务逻辑分析

#### 插件系统架构
- **自包含设计**: 每个插件包含完整的页面、组件、样式
- **核心插件**: 
  - E-commerce: `seckill`, `coupon`, `distribution`, `wallet`
  - Content: `blog`, `ask`, `article`
  - Store Management: `shop`, `realstore`
  - Member System: `membershiplevelvip`, `signin`, `points`

#### DIY可视化系统
- **40+组件**: Layout、Content、E-commerce、Data等类型
- **组件类型**:
  - 基础: `carousel`, `tabs`, `header`, `rich-text`
  - 商品: `goods-list`, `goods-magic`, `seckill`
  - 数据: `data-magic`, `search`, `notice`

---

## 第二部分：Next.js 15重构技术栈

### 2.1 技术栈选择

```json
{
  "核心框架": "Next.js 15 (App Router)",
  "前端库": "React 18 + TypeScript",
  "状态管理": "Zustand + TanStack Query",
  "认证系统": "NextAuth.js v5",
  "样式方案": "Tailwind CSS + CSS Variables",
  "UI组件": "Radix UI + Shadcn/ui",
  "国际化": "next-intl",
  "数据库": "Prisma + PostgreSQL",
  "缓存": "Redis + Next.js Cache",
  "实时通信": "Socket.io",
  "测试": "Jest + Testing Library + Playwright",
  "部署": "Docker + Kubernetes / Vercel"
}
```

### 2.2 项目结构设计

```
shopxo-nextjs/
├── app/                          # Next.js 15 App Router
│   ├── [locale]/                 # 国际化路由
│   │   ├── (auth)/              # 认证相关页面组
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (shop)/              # 主应用页面组
│   │   │   ├── page.tsx         # 首页
│   │   │   ├── category/
│   │   │   ├── product/
│   │   │   ├── cart/
│   │   │   └── user/
│   │   ├── (admin)/             # 管理后台页面组
│   │   └── plugins/             # 插件页面
│   │       ├── seckill/
│   │       ├── coupon/
│   │       └── [plugin]/
│   ├── api/                     # API Routes
│   │   ├── auth/
│   │   ├── products/
│   │   └── plugins/
│   ├── globals.css
│   └── layout.tsx
├── components/                   # React组件
│   ├── ui/                      # 基础UI组件
│   ├── layout/                  # 布局组件
│   ├── diy/                     # DIY可视化组件
│   └── plugins/                 # 插件组件
├── lib/                         # 工具库
│   ├── auth.ts                  # NextAuth配置
│   ├── db.ts                    # 数据库配置
│   ├── theme.ts                 # 主题系统
│   └── utils.ts
├── store/                       # 状态管理
│   ├── auth.ts
│   ├── cart.ts
│   └── theme.ts
├── hooks/                       # 自定义Hooks
├── types/                       # TypeScript类型
├── tests/                       # 测试文件
├── docs/                        # 技术文档
└── middleware.ts                # 中间件
```

---

## 第三部分：核心系统实现详解

### 3.1 主题系统重构

#### 主题管理器实现
```typescript
// lib/theme.ts
export const themes = {
  red: { 
    primary: '#ff0036',
    primaryForeground: '#ffffff',
    secondary: '#fef2f2',
    accent: '#dc2626',
    background: '#ffffff',
    foreground: '#0f172a',
    muted: '#f8fafc',
    border: '#e2e8f0',
  },
  blue: { 
    primary: '#1677ff',
    primaryForeground: '#ffffff',
    secondary: '#eff6ff',
    accent: '#2563eb',
    background: '#ffffff',
    foreground: '#0f172a',
    muted: '#f8fafc',
    border: '#e2e8f0',
  },
  // ... 其他6种主题配置
} as const;

export type ThemeKey = keyof typeof themes;

// 主题CSS变量生成
export function generateThemeCSS(theme: ThemeKey): string {
  const colors = themes[theme];
  return Object.entries(colors)
    .map(([key, value]) => `--${key}: ${value};`)
    .join('\n');
}
```

#### Zustand主题状态管理
```typescript
// store/theme.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ThemeStore {
  currentTheme: ThemeKey;
  isDarkMode: boolean;
  setTheme: (theme: ThemeKey) => void;
  toggleDarkMode: () => void;
  getThemeColors: () => typeof themes[ThemeKey];
}

export const useThemeStore = create<ThemeStore>()(
  persist(
    (set, get) => ({
      currentTheme: 'red',
      isDarkMode: false,
      
      setTheme: (theme) => {
        set({ currentTheme: theme });
        // 动态更新CSS变量
        const root = document.documentElement;
        const colors = themes[theme];
        Object.entries(colors).forEach(([key, value]) => {
          root.style.setProperty(`--${key}`, value);
        });
      },
      
      toggleDarkMode: () => {
        set((state) => ({ isDarkMode: !state.isDarkMode }));
        document.documentElement.classList.toggle('dark');
      },
      
      getThemeColors: () => themes[get().currentTheme],
    }),
    { 
      name: 'theme-storage',
      partialize: (state) => ({ 
        currentTheme: state.currentTheme,
        isDarkMode: state.isDarkMode 
      }),
    }
  )
);
```

#### 主题提供者组件
```typescript
// components/theme/ThemeProvider.tsx
'use client';
import { useEffect } from 'react';
import { useThemeStore } from '@/store/theme';

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const { currentTheme, isDarkMode, setTheme } = useThemeStore();

  useEffect(() => {
    // 初始化主题
    setTheme(currentTheme);
    
    // 应用暗色模式
    if (isDarkMode) {
      document.documentElement.classList.add('dark');
    }
  }, []);

  return (
    <div 
      className={`theme-${currentTheme} ${isDarkMode ? 'dark' : ''}`}
      data-theme={currentTheme}
    >
      {children}
    </div>
  );
}
```

### 3.2 NextAuth.js v5 认证系统

#### 认证配置
```typescript
// lib/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import WeChat from 'next-auth/providers/wechat';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/db';
import bcrypt from 'bcryptjs';

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' },
  
  providers: [
    Credentials({
      name: 'credentials',
      credentials: {
        phone: { label: '手机号', type: 'text' },
        password: { label: '密码', type: 'password' }
      },
      async authorize(credentials) {
        if (!credentials?.phone || !credentials?.password) return null;

        // 验证用户凭据
        const user = await prisma.user.findUnique({
          where: { phone: credentials.phone as string },
          include: { profile: true }
        });

        if (!user || !user.password) return null;

        const isValidPassword = await bcrypt.compare(
          credentials.password as string,
          user.password
        );

        if (!isValidPassword) return null;

        return {
          id: user.id,
          name: user.nickname || user.username,
          email: user.email,
          image: user.avatar,
          role: user.role,
        };
      },
    }),
    
    WeChat({
      clientId: process.env.WECHAT_APP_ID!,
      clientSecret: process.env.WECHAT_APP_SECRET!,
      authorization: {
        params: {
          scope: 'snsapi_userinfo',
        },
      },
    }),
  ],

  pages: {
    signIn: '/login',
    signUp: '/register',
    error: '/auth/error',
  },

  callbacks: {
    async jwt({ token, user, account }) {
      // 首次登录时保存用户信息
      if (user) {
        token.userId = user.id;
        token.role = user.role;
      }

      // 微信登录处理
      if (account?.provider === 'wechat') {
        token.wechatOpenId = account.providerAccountId;
      }

      return token;
    },

    async session({ session, token }) {
      if (token) {
        session.user.id = token.userId as string;
        session.user.role = token.role as string;
      }
      return session;
    },

    async signIn({ user, account, profile }) {
      // 微信登录时的额外处理
      if (account?.provider === 'wechat') {
        await handleWeChatLogin(user, account, profile);
      }
      return true;
    },
  },

  events: {
    async signIn({ user, account }) {
      // 记录登录日志
      await prisma.loginLog.create({
        data: {
          userId: user.id,
          provider: account?.provider || 'credentials',
          ip: '', // 从request获取
          userAgent: '', // 从request获取
        },
      });
    },
  },
});

// 微信登录处理函数
async function handleWeChatLogin(user: any, account: any, profile: any) {
  // 检查是否已存在微信绑定
  const existingUser = await prisma.user.findUnique({
    where: { wechatOpenId: account.providerAccountId },
  });

  if (!existingUser) {
    // 创建新用户或绑定现有用户
    await prisma.user.upsert({
      where: { id: user.id },
      update: {
        wechatOpenId: account.providerAccountId,
        wechatUnionId: profile?.unionid,
      },
      create: {
        wechatOpenId: account.providerAccountId,
        wechatUnionId: profile?.unionid,
        nickname: profile?.nickname,
        avatar: profile?.headimgurl,
      },
    });
  }
}
```

#### 认证中间件
```typescript
// middleware.ts
import { auth } from '@/lib/auth';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

// 需要认证的路由
const protectedRoutes = [
  '/user',
  '/cart',
  '/checkout',
  '/orders',
  '/admin',
];

// 公开路由
const publicRoutes = [
  '/',
  '/login',
  '/register',
  '/products',
  '/category',
  '/api/auth',
];

export default auth((req) => {
  const { nextUrl } = req;
  const isLoggedIn = !!req.auth;

  // 检查是否为受保护的路由
  const isProtectedRoute = protectedRoutes.some(route => 
    nextUrl.pathname.startsWith(route)
  );

  // 检查是否为公开路由
  const isPublicRoute = publicRoutes.some(route => 
    nextUrl.pathname.startsWith(route) || nextUrl.pathname === route
  );

  // 未登录用户访问受保护路由
  if (isProtectedRoute && !isLoggedIn) {
    const callbackUrl = encodeURIComponent(nextUrl.pathname + nextUrl.search);
    return NextResponse.redirect(
      new URL(`/login?callbackUrl=${callbackUrl}`, nextUrl)
    );
  }

  // 已登录用户访问登录页面
  if (isLoggedIn && nextUrl.pathname.startsWith('/login')) {
    return NextResponse.redirect(new URL('/', nextUrl));
  }

  return NextResponse.next();
});

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};
```

#### 认证Hook
```typescript
// hooks/use-auth.ts
import { useSession, signIn, signOut } from 'next-auth/react';
import { useState } from 'react';
import { toast } from '@/components/ui/use-toast';

export function useAuth() {
  const { data: session, status } = useSession();
  const [isLoading, setIsLoading] = useState(false);

  const login = async (credentials: { phone: string; password: string }) => {
    setIsLoading(true);
    try {
      const result = await signIn('credentials', {
        ...credentials,
        redirect: false,
      });

      if (result?.error) {
        toast({
          title: '登录失败',
          description: '手机号或密码错误',
          variant: 'destructive',
        });
        return false;
      }

      toast({
        title: '登录成功',
        description: '欢迎回来！',
      });
      return true;
    } catch (error) {
      toast({
        title: '登录失败',
        description: '网络错误，请稍后重试',
        variant: 'destructive',
      });
      return false;
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    try {
      await signOut({ callbackUrl: '/' });
      toast({
        title: '已退出登录',
        description: '感谢您的使用',
      });
    } catch (error) {
      toast({
        title: '退出失败',
        description: '请稍后重试',
        variant: 'destructive',
      });
    }
  };

  const loginWithWeChat = async () => {
    setIsLoading(true);
    try {
      await signIn('wechat', { callbackUrl: '/' });
    } catch (error) {
      toast({
        title: '微信登录失败',
        description: '请稍后重试',
        variant: 'destructive',
      });
    } finally {
      setIsLoading(false);
    }
  };

  return {
    user: session?.user,
    isAuthenticated: !!session,
    isLoading: status === 'loading' || isLoading,
    login,
    logout,
    loginWithWeChat,
  };
}
```

### 3.3 数据库设计与Prisma模型

#### 完整Schema设计
```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 用户系统
model User {
  id            String    @id @default(cuid())
  phone         String?   @unique
  email         String?   @unique
  username      String?   @unique
  nickname      String?
  avatar        String?
  password      String?
  wechatOpenId  String?   @unique @map("wechat_open_id")
  wechatUnionId String?   @map("wechat_union_id")
  role          UserRole  @default(USER)
  status        UserStatus @default(ACTIVE)
  emailVerified DateTime? @map("email_verified")
  phoneVerified DateTime? @map("phone_verified")
  lastLoginAt   DateTime? @map("last_login_at")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  // 关联关系
  profile       UserProfile?
  addresses     Address[]
  orders        Order[]
  cartItems     CartItem[]
  favorites     Favorite[]
  reviews       Review[]
  points        PointTransaction[]
  loginLogs     LoginLog[]
  
  @@map("users")
}

enum UserRole {
  USER
  ADMIN
  SUPER_ADMIN
  SHOP_OWNER
}

enum UserStatus {
  ACTIVE
  INACTIVE
  BANNED
  PENDING_VERIFICATION
}

model UserProfile {
  id          String    @id @default(cuid())
  userId      String    @unique @map("user_id")
  realName    String?   @map("real_name")
  idCard      String?   @map("id_card")
  birthday    DateTime?
  gender      Gender?
  bio         String?
  level       UserLevel @default(BRONZE)
  experience  Int       @default(0)
  balance     Decimal   @default(0) @db.Decimal(10, 2)
  points      Int       @default(0)
  
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("user_profiles")
}

enum Gender {
  MALE
  FEMALE
  OTHER
}

enum UserLevel {
  BRONZE
  SILVER
  GOLD
  PLATINUM
  DIAMOND
}

// 商品系统
model Category {
  id            String    @id @default(cuid())
  name          String
  slug          String    @unique
  description   String?
  image         String?
  icon          String?
  parentId      String?   @map("parent_id")
  level         Int       @default(1)
  sort          Int       @default(0)
  isActive      Boolean   @default(true) @map("is_active")
  seoTitle      String?   @map("seo_title")
  seoKeywords   String?   @map("seo_keywords")
  seoDescription String?  @map("seo_description")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  parent        Category? @relation("CategoryHierarchy", fields: [parentId], references: [id])
  children      Category[] @relation("CategoryHierarchy")
  products      Product[]
  
  @@map("categories")
}

model Brand {
  id          String    @id @default(cuid())
  name        String    @unique
  logo        String?
  description String?
  website     String?
  isActive    Boolean   @default(true) @map("is_active")
  sort        Int       @default(0)
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  products    Product[]
  
  @@map("brands")
}

model Product {
  id            String    @id @default(cuid())
  title         String
  subtitle      String?
  description   String?
  content       String?   // 富文本详情
  images        Json      // 图片数组
  video         String?   // 视频链接
  price         Decimal   @db.Decimal(10, 2)
  originalPrice Decimal?  @db.Decimal(10, 2) @map("original_price")
  costPrice     Decimal?  @db.Decimal(10, 2) @map("cost_price")
  stock         Int       @default(0)
  minStock      Int       @default(0) @map("min_stock") // 库存预警
  sales         Int       @default(0) // 销量
  virtualSales  Int       @default(0) @map("virtual_sales") // 虚拟销量
  weight        Decimal?  @db.Decimal(8, 3) // 重量(kg)
  volume        Decimal?  @db.Decimal(8, 3) // 体积(m³)
  status        ProductStatus @default(ACTIVE)
  categoryId    String    @map("category_id")
  brandId       String?   @map("brand_id")
  shopId        String?   @map("shop_id")
  
  // SEO相关
  seoTitle      String?   @map("seo_title")
  seoKeywords   String?   @map("seo_keywords")
  seoDescription String?  @map("seo_description")
  
  // 商品属性
  specifications Json?    // 规格参数 {"颜色": ["红色", "蓝色"], "尺寸": ["S", "M", "L"]}
  attributes    Json?     // 商品属性 {"材质": "棉", "产地": "中国"}
  
  // 时间字段
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  publishedAt   DateTime? @map("published_at")
  
  // 关联关系
  category      Category  @relation(fields: [categoryId], references: [id])
  brand         Brand?    @relation(fields: [brandId], references: [id])
  shop          Shop?     @relation(fields: [shopId], references: [id])
  skus          ProductSku[]
  cartItems     CartItem[]
  orderItems    OrderItem[]
  favorites     Favorite[]
  reviews       Review[]
  tags          ProductTag[]
  
  @@map("products")
}

enum ProductStatus {
  DRAFT       // 草稿
  ACTIVE      // 上架
  INACTIVE    // 下架
  OUT_OF_STOCK // 缺货
  DISCONTINUED // 停产
}

model ProductSku {
  id            String    @id @default(cuid())
  productId     String    @map("product_id")
  skuCode       String    @unique @map("sku_code")
  barcode       String?   // 条形码
  specifications Json     // 规格组合 {"颜色": "红色", "尺寸": "L"}
  price         Decimal   @db.Decimal(10, 2)
  originalPrice Decimal?  @db.Decimal(10, 2) @map("original_price")
  costPrice     Decimal?  @db.Decimal(10, 2) @map("cost_price")
  stock         Int       @default(0)
  image         String?   // SKU专属图片
  weight        Decimal?  @db.Decimal(8, 3)
  volume        Decimal?  @db.Decimal(8, 3)
  status        ProductStatus @default(ACTIVE)
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  product       Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  cartItems     CartItem[]
  orderItems    OrderItem[]
  
  @@map("product_skus")
}

model ProductTag {
  id          String    @id @default(cuid())
  productId   String    @map("product_id")
  name        String
  color       String?   // 标签颜色
  sort        Int       @default(0)
  
  product     Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  
  @@unique([productId, name])
  @@map("product_tags")
}

// 购物车系统
model CartItem {
  id            String    @id @default(cuid())
  userId        String    @map("user_id")
  productId     String    @map("product_id")
  skuId         String?   @map("sku_id")
  quantity      Int       @default(1)
  price         Decimal   @db.Decimal(10, 2) // 加入购物车时的价格
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  user          User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  product       Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  sku           ProductSku? @relation(fields: [skuId], references: [id], onDelete: Cascade)
  
  @@unique([userId, productId, skuId])
  @@map("cart_items")
}

// 订单系统
model Order {
  id            String      @id @default(cuid())
  orderNo       String      @unique @map("order_no")
  userId        String      @map("user_id")
  shopId        String?     @map("shop_id")
  type          OrderType   @default(NORMAL)
  status        OrderStatus @default(PENDING_PAYMENT)
  paymentStatus PaymentStatus @default(PENDING)
  paymentMethod String?     @map("payment_method")
  
  // 金额信息
  totalAmount   Decimal     @db.Decimal(10, 2) @map("total_amount")
  discountAmount Decimal    @default(0) @db.Decimal(10, 2) @map("discount_amount")
  shippingFee   Decimal     @default(0) @db.Decimal(10, 2) @map("shipping_fee")
  taxAmount     Decimal     @default(0) @db.Decimal(10, 2) @map("tax_amount")
  actualAmount  Decimal     @db.Decimal(10, 2) @map("actual_amount")
  
  // 收货信息
  receiverName    String    @map("receiver_name")
  receiverPhone   String    @map("receiver_phone")
  receiverAddress String    @map("receiver_address")
  receiverZipCode String?   @map("receiver_zip_code")
  
  // 备注信息
  buyerMessage    String?   @map("buyer_message")
  sellerMessage   String?   @map("seller_message")
  adminMessage    String?   @map("admin_message")
  
  // 时间字段
  createdAt       DateTime  @default(now()) @map("created_at")
  paidAt          DateTime? @map("paid_at")
  shippedAt       DateTime? @map("shipped_at")
  completedAt     DateTime? @map("completed_at")
  canceledAt      DateTime? @map("canceled_at")
  refundedAt      DateTime? @map("refunded_at")
  
  // 关联关系
  user            User        @relation(fields: [userId], references: [id])
  shop            Shop?       @relation(fields: [shopId], references: [id])
  items           OrderItem[]
  payments        Payment[]
  logistics       OrderLogistics[]
  statusHistory   OrderStatusHistory[]
  
  @@map("orders")
}

enum OrderType {
  NORMAL      // 普通订单
  SECKILL     // 秒杀订单
  GROUP_BUY   // 团购订单
  PRESALE     // 预售订单
}

enum OrderStatus {
  PENDING_PAYMENT // 待付款
  PAID           // 已付款
  SHIPPED        // 已发货
  COMPLETED      // 已完成
  CANCELED       // 已取消
  REFUNDING      // 退款中
  REFUNDED       // 已退款
}

enum PaymentStatus {
  PENDING
  SUCCESS
  FAILED
  REFUNDED
  PARTIAL_REFUNDED
}

model OrderItem {
  id            String    @id @default(cuid())
  orderId       String    @map("order_id")
  productId     String    @map("product_id")
  skuId         String?   @map("sku_id")
  productTitle  String    @map("product_title")
  productImage  String    @map("product_image")
  skuSpecs      Json?     @map("sku_specs") // SKU规格信息快照
  price         Decimal   @db.Decimal(10, 2)
  quantity      Int
  totalAmount   Decimal   @db.Decimal(10, 2) @map("total_amount")
  
  order         Order     @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product       Product   @relation(fields: [productId], references: [id])
  sku           ProductSku? @relation(fields: [skuId], references: [id])
  
  @@map("order_items")
}

model OrderStatusHistory {
  id          String      @id @default(cuid())
  orderId     String      @map("order_id")
  fromStatus  OrderStatus? @map("from_status")
  toStatus    OrderStatus @map("to_status")
  note        String?
  operatorId  String?     @map("operator_id")
  createdAt   DateTime    @default(now()) @map("created_at")
  
  order       Order       @relation(fields: [orderId], references: [id], onDelete: Cascade)
  
  @@map("order_status_history")
}

// 支付系统
model Payment {
  id              String        @id @default(cuid())
  orderId         String        @map("order_id")
  paymentNo       String        @unique @map("payment_no")
  method          PaymentMethod
  provider        String        // 支付提供商：wechat, alipay, stripe等
  amount          Decimal       @db.Decimal(10, 2)
  currency        String        @default("CNY")
  status          PaymentStatus @default(PENDING)
  
  // 第三方支付信息
  providerOrderId String?       @map("provider_order_id")
  providerResponse Json?        @map("provider_response")
  
  // 时间字段
  createdAt       DateTime      @default(now()) @map("created_at")
  paidAt          DateTime?     @map("paid_at")
  failedAt        DateTime?     @map("failed_at")
  refundedAt      DateTime?     @map("refunded_at")
  
  order           Order         @relation(fields: [orderId], references: [id])
  
  @@map("payments")
}

enum PaymentMethod {
  WECHAT_PAY
  ALIPAY
  CREDIT_CARD
  BANK_TRANSFER
  BALANCE
  POINTS
}

// 物流系统
model OrderLogistics {
  id            String    @id @default(cuid())
  orderId       String    @map("order_id")
  company       String    // 物流公司
  trackingNo    String    @map("tracking_no") // 运单号
  status        LogisticsStatus @default(PENDING)
  currentLocation String? @map("current_location")
  estimatedDelivery DateTime? @map("estimated_delivery")
  actualDelivery DateTime? @map("actual_delivery")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  order         Order     @relation(fields: [orderId], references: [id])
  tracks        LogisticsTrack[]
  
  @@map("order_logistics")
}

enum LogisticsStatus {
  PENDING     // 待发货
  PICKED_UP   // 已揽收
  IN_TRANSIT  // 运输中
  OUT_FOR_DELIVERY // 派送中
  DELIVERED   // 已送达
  FAILED      // 派送失败
  RETURNED    // 已退回
}

model LogisticsTrack {
  id          String          @id @default(cuid())
  logisticsId String          @map("logistics_id")
  location    String
  description String
  timestamp   DateTime
  createdAt   DateTime        @default(now()) @map("created_at")
  
  logistics   OrderLogistics  @relation(fields: [logisticsId], references: [id])
  
  @@map("logistics_tracks")
}

// 地址系统
model Address {
  id          String    @id @default(cuid())
  userId      String    @map("user_id")
  name        String    // 收货人姓名
  phone       String    // 联系电话
  province    String    // 省份
  city        String    // 城市
  district    String    // 区县
  street      String    // 详细地址
  zipCode     String?   @map("zip_code")
  isDefault   Boolean   @default(false) @map("is_default")
  tag         String?   // 地址标签：家、公司等
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("addresses")
}

// 收藏夹
model Favorite {
  id          String    @id @default(cuid())
  userId      String    @map("user_id")
  productId   String    @map("product_id")
  createdAt   DateTime  @default(now()) @map("created_at")
  
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  product     Product   @relation(fields: [productId], references: [id], onDelete: Cascade)
  
  @@unique([userId, productId])
  @@map("favorites")
}

// 评价系统
model Review {
  id          String      @id @default(cuid())
  userId      String      @map("user_id")
  productId   String      @map("product_id")
  orderId     String      @map("order_id")
  rating      Int         // 评分 1-5
  content     String?     // 评价内容
  images      Json?       // 评价图片
  isAnonymous Boolean     @default(false) @map("is_anonymous")
  status      ReviewStatus @default(PENDING)
  
  // 商家回复
  reply       String?
  repliedAt   DateTime?   @map("replied_at")
  
  // 有用性投票
  helpfulCount Int        @default(0) @map("helpful_count")
  
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  
  user        User        @relation(fields: [userId], references: [id])
  product     Product     @relation(fields: [productId], references: [id])
  
  @@unique([userId, productId, orderId])
  @@map("reviews")
}

enum ReviewStatus {
  PENDING
  APPROVED
  REJECTED
}

// 积分系统
model PointTransaction {
  id          String          @id @default(cuid())
  userId      String          @map("user_id")
  type        PointType
  amount      Int             // 正数为获得，负数为消费
  balance     Int             // 交易后余额
  description String
  relatedId   String?         @map("related_id") // 关联订单ID等
  relatedType String?         @map("related_type") // 关联类型
  createdAt   DateTime        @default(now()) @map("created_at")
  
  user        User            @relation(fields: [userId], references: [id])
  
  @@map("point_transactions")
}

enum PointType {
  REGISTER        // 注册奖励
  LOGIN           // 登录奖励
  ORDER_COMPLETE  // 订单完成
  REVIEW          // 评价奖励
  REFERRAL        // 推荐奖励
  CONSUME         // 积分消费
  EXPIRE          // 积分过期
  ADMIN_ADJUST    // 管理员调整
}

// 多商户系统
model Shop {
  id          String      @id @default(cuid())
  name        String
  slug        String      @unique
  logo        String?
  banner      String?
  description String?
  ownerId     String      @map("owner_id")
  status      ShopStatus  @default(PENDING)
  
  // 联系信息
  phone       String?
  email       String?
  address     String?
  
  // 营业信息
  businessHours Json?     @map("business_hours") // {"monday": {"open": "09:00", "close": "18:00"}}
  
  // 统计信息
  productCount Int        @default(0) @map("product_count")
  orderCount   Int        @default(0) @map("order_count")
  rating       Decimal?   @db.Decimal(3, 2)
  
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  
  products    Product[]
  orders      Order[]
  
  @@map("shops")
}

enum ShopStatus {
  PENDING     // 待审核
  APPROVED    // 已通过
  REJECTED    // 已拒绝
  SUSPENDED   // 已暂停
  CLOSED      // 已关闭
}

// DIY页面系统
model DiyPage {
  id          String        @id @default(cuid())
  name        String
  type        DiyPageType
  config      Json          // 页面全局配置
  components  Json          // 组件配置数组
  isDefault   Boolean       @default(false) @map("is_default")
  isActive    Boolean       @default(true) @map("is_active")
  createdAt   DateTime      @default(now()) @map("created_at")
  updatedAt   DateTime      @updatedAt @map("updated_at")
  
  @@map("diy_pages")
}

enum DiyPageType {
  HOME        // 首页
  CATEGORY    // 分类页
  CUSTOM      // 自定义页面
  SHOP        // 商户页面
}

// 插件系统
model Plugin {
  id          String    @id @default(cuid())
  name        String    @unique
  version     String
  description String?
  author      String?
  isEnabled   Boolean   @default(true) @map("is_enabled")
  config      Json?     // 插件配置
  permissions Json?     // 权限配置
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  @@map("plugins")
}

// 系统配置
model SystemConfig {
  id          String    @id @default(cuid())
  key         String    @unique
  value       Json
  description String?
  group       String?   // 配置分组
  isPublic    Boolean   @default(false) @map("is_public") // 是否为公开配置
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  @@map("system_configs")
}

// 登录日志
model LoginLog {
  id          String    @id @default(cuid())
  userId      String    @map("user_id")
  provider    String    // 登录方式
  ip          String
  userAgent   String    @map("user_agent")
  location    String?   // 登录地点
  deviceInfo  Json?     @map("device_info")
  createdAt   DateTime  @default(now()) @map("created_at")
  
  user        User      @relation(fields: [userId], references: [id])
  
  @@map("login_logs")
}

// 操作日志
model OperationLog {
  id          String    @id @default(cuid())
  userId      String?   @map("user_id")
  action      String    // 操作类型
  resource    String    // 操作资源
  resourceId  String?   @map("resource_id")
  details     Json?     // 操作详情
  ip          String
  userAgent   String    @map("user_agent")
  createdAt   DateTime  @default(now()) @map("created_at")
  
  @@map("operation_logs")
}
```

### 3.4 插件系统架构

#### 插件注册机制
```typescript
// lib/plugin-system.ts
export interface PluginConfig {
  id: string;
  name: string;
  version: string;
  description?: string;
  author?: string;
  dependencies?: string[];
  permissions?: PluginPermission[];
  routes?: PluginRoute[];
  components?: PluginComponent[];
  hooks?: PluginHook[];
  settings?: PluginSetting[];
}

export interface PluginRoute {
  path: string;
  component: string;
  meta?: {
    title?: string;
    requireAuth?: boolean;
    permissions?: string[];
  };
}

export interface PluginComponent {
  name: string;
  component: React.ComponentType<any>;
  category?: 'layout' | 'business' | 'display' | 'form';
  icon?: string;
  description?: string;
}

export interface PluginHook {
  name: string;
  handler: (...args: any[]) => any;
  priority?: number;
}

export interface PluginPermission {
  name: string;
  description: string;
  group?: string;
}

export interface PluginSetting {
  key: string;
  type: 'string' | 'number' | 'boolean' | 'select' | 'textarea';
  label: string;
  description?: string;
  defaultValue?: any;
  options?: Array<{ label: string; value: any }>;
  required?: boolean;
}

class PluginRegistry {
  private plugins = new Map<string, PluginConfig>();
  private hooks = new Map<string, PluginHook[]>();
  private components = new Map<string, PluginComponent>();

  register(config: PluginConfig): void {
    // 检查依赖
    if (config.dependencies) {
      const missingDeps = config.dependencies.filter(dep => !this.plugins.has(dep));
      if (missingDeps.length > 0) {
        throw new Error(`Plugin ${config.id} missing dependencies: ${missingDeps.join(', ')}`);
      }
    }

    // 注册插件
    this.plugins.set(config.id, config);

    // 注册路由
    if (config.routes) {
      config.routes.forEach(route => {
        this.registerRoute(config.id, route);
      });
    }

    // 注册组件
    if (config.components) {
      config.components.forEach(component => {
        this.components.set(component.name, component);
      });
    }

    // 注册钩子
    if (config.hooks) {
      config.hooks.forEach(hook => {
        this.registerHook(hook.name, hook);
      });
    }

    console.log(`Plugin ${config.id} registered successfully`);
  }

  unregister(pluginId: string): void {
    const config = this.plugins.get(pluginId);
    if (!config) return;

    // 移除组件
    if (config.components) {
      config.components.forEach(component => {
        this.components.delete(component.name);
      });
    }

    // 移除钩子
    if (config.hooks) {
      config.hooks.forEach(hook => {
        this.unregisterHook(hook.name, hook);
      });
    }

    this.plugins.delete(pluginId);
  }

  getPlugin(id: string): PluginConfig | undefined {
    return this.plugins.get(id);
  }

  getComponent(name: string): PluginComponent | undefined {
    return this.components.get(name);
  }

  private registerRoute(pluginId: string, route: PluginRoute): void {
    // 动态路由注册逻辑
    // 在Next.js中，这可能需要配合动态导入和路由配置
  }

  private registerHook(name: string, hook: PluginHook): void {
    if (!this.hooks.has(name)) {
      this.hooks.set(name, []);
    }
    
    const hooks = this.hooks.get(name)!;
    hooks.push(hook);
    
    // 按优先级排序
    hooks.sort((a, b) => (b.priority || 0) - (a.priority || 0));
  }

  private unregisterHook(name: string, hook: PluginHook): void {
    const hooks = this.hooks.get(name);
    if (hooks) {
      const index = hooks.indexOf(hook);
      if (index > -1) {
        hooks.splice(index, 1);
      }
    }
  }

  async executeHook(name: string, ...args: any[]): Promise<any[]> {
    const hooks = this.hooks.get(name) || [];
    const results = [];

    for (const hook of hooks) {
      try {
        const result = await hook.handler(...args);
        results.push(result);
      } catch (error) {
        console.error(`Hook ${name} execution failed:`, error);
      }
    }

    return results;
  }

  getEnabledPlugins(): PluginConfig[] {
    return Array.from(this.plugins.values()).filter(plugin => {
      // 检查插件是否启用（从数据库或配置中获取）
      return true; // 简化示例
    });
  }
}

export const pluginRegistry = new PluginRegistry();
```

#### 插件开发示例 - 秒杀插件
```typescript
// plugins/seckill/config.ts
import { PluginConfig } from '@/lib/plugin-system';
import { SeckillList } from './components/SeckillList';
import { SeckillTimer } from './components/SeckillTimer';
import { SeckillButton } from './components/SeckillButton';

export const seckillPluginConfig: PluginConfig = {
  id: 'seckill',
  name: '秒杀活动',
  version: '1.0.0',
  description: '商品秒杀活动管理',
  author: 'ShopXO Team',
  
  permissions: [
    { name: 'seckill:view', description: '查看秒杀活动' },
    { name: 'seckill:manage', description: '管理秒杀活动' },
  ],
  
  routes: [
    {
      path: '/plugins/seckill',
      component: 'SeckillIndex',
      meta: {
        title: '秒杀活动',
        requireAuth: false,
      },
    },
    {
      path: '/plugins/seckill/manage',
      component: 'SeckillManage',
      meta: {
        title: '秒杀管理',
        requireAuth: true,
        permissions: ['seckill:manage'],
      },
    },
  ],
  
  components: [
    {
      name: 'SeckillList',
      component: SeckillList,
      category: 'business',
      icon: 'zap',
      description: '秒杀商品列表',
    },
    {
      name: 'SeckillTimer',
      component: SeckillTimer,
      category: 'display',
      icon: 'clock',
      description: '秒杀倒计时',
    },
    {
      name: 'SeckillButton',
      component: SeckillButton,
      category: 'business',
      icon: 'shopping-cart',
      description: '秒杀购买按钮',
    },
  ],
  
  hooks: [
    {
      name: 'beforeAddToCart',
      handler: async (productId: string, quantity: number) => {
        // 检查是否为秒杀商品
        const seckillProduct = await checkSeckillProduct(productId);
        if (seckillProduct) {
          return validateSeckillPurchase(seckillProduct, quantity);
        }
        return true;
      },
      priority: 10,
    },
    {
      name: 'orderCreated',
      handler: async (order: any) => {
        // 处理秒杀订单
        await handleSeckillOrder(order);
      },
      priority: 5,
    },
  ],
  
  settings: [
    {
      key: 'max_purchase_quantity',
      type: 'number',
      label: '最大购买数量',
      description: '单个用户最大购买数量限制',
      defaultValue: 5,
      required: true,
    },
    {
      key: 'auto_start',
      type: 'boolean',
      label: '自动开始',
      description: '活动时间到达时自动开始',
      defaultValue: true,
    },
  ],
};

async function checkSeckillProduct(productId: string) {
  // 检查商品是否参与秒杀活动
  return await prisma.seckillProduct.findFirst({
    where: {
      productId,
      startTime: { lte: new Date() },
      endTime: { gte: new Date() },
      isActive: true,
    },
  });
}

async function validateSeckillPurchase(seckillProduct: any, quantity: number) {
  // 验证秒杀购买条件
  if (quantity > seckillProduct.maxQuantity) {
    throw new Error(`超出最大购买数量限制：${seckillProduct.maxQuantity}`);
  }
  
  if (seckillProduct.currentStock < quantity) {
    throw new Error('库存不足');
  }
  
  return true;
}

async function handleSeckillOrder(order: any) {
  // 处理秒杀订单逻辑
  // 更新库存、记录购买记录等
}
```

#### 插件组件渲染器
```typescript
// components/plugins/PluginRenderer.tsx
'use client';
import { useEffect, useState } from 'react';
import { pluginRegistry } from '@/lib/plugin-system';
import { logger } from '@/lib/logger';

interface PluginRendererProps {
  componentName: string;
  props?: Record<string, any>;
  fallback?: React.ReactNode;
}

export function PluginRenderer({ 
  componentName, 
  props = {}, 
  fallback 
}: PluginRendererProps) {
  const [Component, setComponent] = useState<React.ComponentType<any> | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadComponent = async () => {
      try {
        const pluginComponent = pluginRegistry.getComponent(componentName);
        
        if (!pluginComponent) {
          throw new Error(`Plugin component ${componentName} not found`);
        }

        setComponent(() => pluginComponent.component);
        setError(null);
      } catch (err) {
        const errorMsg = err instanceof Error ? err.message : 'Unknown error';
        setError(errorMsg);
        logger.error('Plugin component load failed', err as Error, {
          componentName,
          props,
        });
      }
    };

    loadComponent();
  }, [componentName]);

  if (error) {
    if (fallback) {
      return <>{fallback}</>;
    }
    
    return (
      <div className="p-4 border border-red-200 bg-red-50 rounded-lg">
        <p className="text-red-600 text-sm">
          插件组件加载失败: {error}
        </p>
      </div>
    );
  }

  if (!Component) {
    return (
      <div className="p-4 bg-gray-100 rounded-lg animate-pulse">
        <div className="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
        <div className="h-3 bg-gray-200 rounded w-1/2"></div>
      </div>
    );
  }

  return <Component {...props} />;
}

// 插件管理页面
// pages/admin/plugins/page.tsx
import { useState, useEffect } from 'react';
import { pluginRegistry } from '@/lib/plugin-system';
import { Button } from '@/components/ui/button';
import { Switch } from '@/components/ui/switch';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

export default function PluginManagePage() {
  const [plugins, setPlugins] = useState(pluginRegistry.getEnabledPlugins());

  const handleTogglePlugin = async (pluginId: string, enabled: boolean) => {
    try {
      if (enabled) {
        // 启用插件逻辑
        await enablePlugin(pluginId);
      } else {
        // 禁用插件逻辑
        await disablePlugin(pluginId);
      }
      
      // 刷新插件列表
      setPlugins(pluginRegistry.getEnabledPlugins());
    } catch (error) {
      console.error('Toggle plugin failed:', error);
    }
  };

  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">插件管理</h1>
      
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {plugins.map((plugin) => (
          <Card key={plugin.id}>
            <CardHeader>
              <CardTitle className="flex items-center justify-between">
                <span>{plugin.name}</span>
                <Switch 
                  checked={true} // 从数据库获取状态
                  onCheckedChange={(enabled) => 
                    handleTogglePlugin(plugin.id, enabled)
                  }
                />
              </CardTitle>
            </CardHeader>
            
            <CardContent>
              <p className="text-sm text-gray-600 mb-4">
                {plugin.description}
              </p>
              
              <div className="space-y-2 text-xs">
                <div>版本: {plugin.version}</div>
                <div>作者: {plugin.author}</div>
                {plugin.components && (
                  <div>组件: {plugin.components.length} 个</div>
                )}
                {plugin.routes && (
                  <div>路由: {plugin.routes.length} 个</div>
                )}
              </div>
              
              <div className="mt-4 flex space-x-2">
                <Button variant="outline" size="sm">
                  配置
                </Button>
                <Button variant="outline" size="sm">
                  详情
                </Button>
              </div>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  );
}

async function enablePlugin(pluginId: string) {
  // 启用插件的数据库操作
  await prisma.plugin.update({
    where: { name: pluginId },
    data: { isEnabled: true },
  });
}

async function disablePlugin(pluginId: string) {
  // 禁用插件的数据库操作
  await prisma.plugin.update({
    where: { name: pluginId },
    data: { isEnabled: false },
  });
}
```

### 3.5 DIY可视化编辑器

#### 组件数据结构
```typescript
// types/diy.ts
export interface DiyComponent {
  id: string;
  type: string;
  name: string;
  props: Record<string, any>;
  style: ComponentStyle;
  children?: DiyComponent[];
  conditions?: ComponentCondition[]; // 显示条件
}

export interface ComponentStyle {
  // 布局
  display?: 'block' | 'inline-block' | 'flex' | 'grid';
  position?: 'static' | 'relative' | 'absolute' | 'fixed';
  width?: string | number;
  height?: string | number;
  
  // 间距
  margin: SpacingValue;
  padding: SpacingValue;
  
  // 边框
  border: BorderStyle;
  borderRadius?: string | number;
  
  // 背景
  background: BackgroundStyle;
  
  // 文字
  fontSize?: string | number;
  fontWeight?: string | number;
  color?: string;
  textAlign?: 'left' | 'center' | 'right';
  
  // 阴影
  boxShadow?: string;
  
  // 变换
  transform?: string;
  
  // 层级
  zIndex?: number;
  
  // 动画
  animation?: AnimationStyle;
}

export interface SpacingValue {
  top?: number;
  right?: number;
  bottom?: number;
  left?: number;
}

export interface BorderStyle {
  width?: number;
  style?: 'solid' | 'dashed' | 'dotted' | 'none';
  color?: string;
}

export interface BackgroundStyle {
  type?: 'color' | 'image' | 'gradient';
  color?: string;
  image?: string;
  gradient?: {
    type: 'linear' | 'radial';
    colors: Array<{ color: string; position: number }>;
    direction?: number;
  };
  size?: 'cover' | 'contain' | 'auto';
  position?: string;
  repeat?: 'no-repeat' | 'repeat' | 'repeat-x' | 'repeat-y';
}

export interface AnimationStyle {
  name?: string;
  duration?: number;
  delay?: number;
  timingFunction?: string;
  iterationCount?: number | 'infinite';
  direction?: 'normal' | 'reverse' | 'alternate';
}

export interface ComponentCondition {
  type: 'device' | 'user' | 'time' | 'custom';
  operator: 'eq' | 'ne' | 'gt' | 'lt' | 'in' | 'contains';
  value: any;
  field?: string;
}
```

#### 可视化编辑器核心
```typescript
// components/diy/DiyEditor.tsx
'use client';
import { useState, useCallback, useRef } from 'react';
import { DndProvider, useDrop, useDrag } from 'react-dnd';
import { HTML5Backend } from 'react-dnd-html5-backend';
import { DiyComponent } from '@/types/diy';
import { ComponentPalette } from './ComponentPalette';
import { PropertyPanel } from './PropertyPanel';
import { CanvasArea } from './CanvasArea';
import { Toolbar } from './Toolbar';
import { useToast } from '@/components/ui/use-toast';

interface DiyEditorProps {
  initialComponents?: DiyComponent[];
  onSave?: (components: DiyComponent[]) => void;
  mode?: 'edit' | 'preview';
}

export function DiyEditor({ 
  initialComponents = [], 
  onSave,
  mode = 'edit' 
}: DiyEditorProps) {
  const [components, setComponents] = useState<DiyComponent[]>(initialComponents);
  const [selectedComponent, setSelectedComponent] = useState<string | null>(null);
  const [previewMode, setPreviewMode] = useState(mode === 'preview');
  const [history, setHistory] = useState<DiyComponent[][]>([initialComponents]);
  const [historyIndex, setHistoryIndex] = useState(0);
  const { toast } = useToast();

  // 添加组件
  const addComponent = useCallback((componentType: string, dropIndex?: number) => {
    const newComponent: DiyComponent = {
      id: `component-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type: componentType,
      name: getComponentDisplayName(componentType),
      props: getDefaultProps(componentType),
      style: getDefaultStyle(componentType),
    };

    setComponents(prev => {
      const newComponents = [...prev];
      if (typeof dropIndex === 'number') {
        newComponents.splice(dropIndex, 0, newComponent);
      } else {
        newComponents.push(newComponent);
      }
      return newComponents;
    });

    // 自动选中新添加的组件
    setSelectedComponent(newComponent.id);
    
    // 保存到历史记录
    saveToHistory();
    
    toast({
      title: '组件已添加',
      description: `${newComponent.name} 组件已添加到画布`,
    });
  }, []);

  // 删除组件
  const deleteComponent = useCallback((componentId: string) => {
    setComponents(prev => prev.filter(comp => comp.id !== componentId));
    
    if (selectedComponent === componentId) {
      setSelectedComponent(null);
    }
    
    saveToHistory();
    
    toast({
      title: '组件已删除',
      description: '组件已从画布中移除',
    });
  }, [selectedComponent]);

  // 复制组件
  const duplicateComponent = useCallback((componentId: string) => {
    const component = components.find(comp => comp.id === componentId);
    if (!component) return;

    const duplicatedComponent: DiyComponent = {
      ...component,
      id: `component-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      name: `${component.name} 副本`,
    };

    setComponents(prev => {
      const index = prev.findIndex(comp => comp.id === componentId);
      const newComponents = [...prev];
      newComponents.splice(index + 1, 0, duplicatedComponent);
      return newComponents;
    });

    setSelectedComponent(duplicatedComponent.id);
    saveToHistory();
  }, [components]);

  // 移动组件
  const moveComponent = useCallback((fromIndex: number, toIndex: number) => {
    setComponents(prev => {
      const newComponents = [...prev];
      const [removed] = newComponents.splice(fromIndex, 1);
      newComponents.splice(toIndex, 0, removed);
      return newComponents;
    });
    
    saveToHistory();
  }, []);

  // 更新组件属性
  const updateComponent = useCallback((componentId: string, updates: Partial<DiyComponent>) => {
    setComponents(prev => 
      prev.map(comp => 
        comp.id === componentId 
          ? { ...comp, ...updates }
          : comp
      )
    );
  }, []);

  // 更新组件样式
  const updateComponentStyle = useCallback((componentId: string, styleUpdates: Partial<ComponentStyle>) => {
    updateComponent(componentId, {
      style: {
        ...components.find(comp => comp.id === componentId)?.style,
        ...styleUpdates,
      },
    });
  }, [components, updateComponent]);

  // 保存到历史记录
  const saveToHistory = useCallback(() => {
    setHistory(prev => {
      const newHistory = prev.slice(0, historyIndex + 1);
      newHistory.push([...components]);
      return newHistory.slice(-50); // 最多保存50个历史记录
    });
    setHistoryIndex(prev => prev + 1);
  }, [components, historyIndex]);

  // 撤销
  const undo = useCallback(() => {
    if (historyIndex > 0) {
      const newIndex = historyIndex - 1;
      setComponents(history[newIndex]);
      setHistoryIndex(newIndex);
    }
  }, [history, historyIndex]);

  // 重做
  const redo = useCallback(() => {
    if (historyIndex < history.length - 1) {
      const newIndex = historyIndex + 1;
      setComponents(history[newIndex]);
      setHistoryIndex(newIndex);
    }
  }, [history, historyIndex]);

  // 保存页面
  const handleSave = useCallback(async () => {
    try {
      await onSave?.(components);
      toast({
        title: '保存成功',
        description: '页面配置已保存',
      });
    } catch (error) {
      toast({
        title: '保存失败',
        description: '请稍后重试',
        variant: 'destructive',
      });
    }
  }, [components, onSave]);

  const selectedComponentData = components.find(comp => comp.id === selectedComponent);

  if (previewMode) {
    return (
      <div className="w-full">
        <CanvasArea 
          components={components}
          selectedComponent={null}
          onSelectComponent={() => {}}
          previewMode={true}
        />
      </div>
    );
  }

  return (
    <DndProvider backend={HTML5Backend}>
      <div className="flex h-screen bg-gray-100">
        {/* 组件面板 */}
        <div className="w-64 bg-white border-r border-gray-200">
          <ComponentPalette onAddComponent={addComponent} />
        </div>

        {/* 主编辑区域 */}
        <div className="flex-1 flex flex-col">
          {/* 工具栏 */}
          <Toolbar
            onSave={handleSave}
            onUndo={undo}
            onRedo={redo}
            onPreview={() => setPreviewMode(true)}
            canUndo={historyIndex > 0}
            canRedo={historyIndex < history.length - 1}
          />

          {/* 画布区域 */}
          <div className="flex-1 overflow-auto bg-gray-50 p-4">
            <CanvasArea
              components={components}
              selectedComponent={selectedComponent}
              onSelectComponent={setSelectedComponent}
              onMoveComponent={moveComponent}
              onDeleteComponent={deleteComponent}
              onDuplicateComponent={duplicateComponent}
            />
          </div>
        </div>

        {/* 属性面板 */}
        <div className="w-80 bg-white border-l border-gray-200">
          <PropertyPanel
            component={selectedComponentData}
            onUpdateComponent={updateComponent}
            onUpdateStyle={updateComponentStyle}
          />
        </div>
      </div>
    </DndProvider>
  );
}

// 获取组件默认属性
function getDefaultProps(componentType: string): Record<string, any> {
  const defaultProps: Record<string, Record<string, any>> = {
    text: {
      content: '文本内容',
      tag: 'p',
    },
    image: {
      src: '/placeholder.jpg',
      alt: '图片',
      fit: 'cover',
    },
    button: {
      text: '按钮',
      type: 'primary',
      link: '',
    },
    carousel: {
      images: [],
      autoplay: true,
      interval: 3000,
    },
    productList: {
      categoryId: '',
      limit: 10,
      layout: 'grid',
      columns: 2,
    },
  };

  return defaultProps[componentType] || {};
}

// 获取组件默认样式
function getDefaultStyle(componentType: string): ComponentStyle {
  const defaultStyles: Record<string, ComponentStyle> = {
    text: {
      margin: { top: 0, right: 0, bottom: 16, left: 0 },
      padding: { top: 0, right: 0, bottom: 0, left: 0 },
      border: { width: 0, style: 'none', color: '#transparent' },
      background: { type: 'color', color: 'transparent' },
      fontSize: 14,
      color: '#333333',
    },
    image: {
      margin: { top: 0, right: 0, bottom: 16, left: 0 },
      padding: { top: 0, right: 0, bottom: 0, left: 0 },
      border: { width: 0, style: 'none', color: 'transparent' },
      background: { type: 'color', color: 'transparent' },
      width: '100%',
      height: 200,
    },
    button: {
      margin: { top: 0, right: 0, bottom: 16, left: 0 },
      padding: { top: 12, right: 24, bottom: 12, left: 24 },
      border: { width: 1, style: 'solid', color: '#1677ff' },
      background: { type: 'color', color: '#1677ff' },
      borderRadius: 6,
      color: '#ffffff',
      textAlign: 'center',
    },
  };

  return defaultStyles[componentType] || {
    margin: { top: 0, right: 0, bottom: 0, left: 0 },
    padding: { top: 0, right: 0, bottom: 0, left: 0 },
    border: { width: 0, style: 'none', color: 'transparent' },
    background: { type: 'color', color: 'transparent' },
  };
}

function getComponentDisplayName(componentType: string): string {
  const names: Record<string, string> = {
    text: '文本',
    image: '图片',
    button: '按钮',
    carousel: '轮播图',
    productList: '商品列表',
    video: '视频',
    divider: '分割线',
    space: '间距',
    tabs: '标签页',
    notice: '公告',
    search: '搜索框',
  };

  return names[componentType] || componentType;
}
```

#### 组件画布区域
```typescript
// components/diy/CanvasArea.tsx
'use client';
import { useDrop } from 'react-dnd';
import { DiyComponent } from '@/types/diy';
import { DiyComponentRenderer } from './DiyComponentRenderer';
import { DropZone } from './DropZone';

interface CanvasAreaProps {
  components: DiyComponent[];
  selectedComponent: string | null;
  onSelectComponent: (id: string | null) => void;
  onMoveComponent?: (fromIndex: number, toIndex: number) => void;
  onDeleteComponent?: (id: string) => void;
  onDuplicateComponent?: (id: string) => void;
  previewMode?: boolean;
}

export function CanvasArea({
  components,
  selectedComponent,
  onSelectComponent,
  onMoveComponent,
  onDeleteComponent,
  onDuplicateComponent,
  previewMode = false,
}: CanvasAreaProps) {
  const [{ isOver }, drop] = useDrop({
    accept: 'component',
    drop: (item: { type: string }, monitor) => {
      if (!monitor.isOver({ shallow: true })) return;
      
      // 在末尾添加组件
      // 这里需要通过回调通知父组件
    },
    collect: (monitor) => ({
      isOver: monitor.isOver({ shallow: true }),
    }),
  });

  return (
    <div 
      ref={drop}
      className={`
        min-h-[600px] bg-white rounded-lg shadow-sm
        ${isOver ? 'bg-blue-50 border-2 border-blue-300 border-dashed' : 'border border-gray-200'}
        ${previewMode ? '' : 'relative'}
      `}
    >
      {/* 手机框架 */}
      <div className={`mx-auto ${previewMode ? 'w-full' : 'w-[375px]'} min-h-[600px] relative`}>
        {components.length === 0 ? (
          <div className="flex items-center justify-center h-64 text-gray-400">
            <div className="text-center">
              <svg className="w-16 h-16 mx-auto mb-4 text-gray-300" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={1} d="M12 6v6m0 0v6m0-6h6m-6 0H6" />
              </svg>
              <p>从左侧拖拽组件到此处开始设计</p>
            </div>
          </div>
        ) : (
          <div className="space-y-1">
            {components.map((component, index) => (
              <div key={component.id} className="relative">
                {!previewMode && (
                  <DropZone
                    index={index}
                    onDrop={(dragIndex, hoverIndex) => {
                      onMoveComponent?.(dragIndex, hoverIndex);
                    }}
                  />
                )}
                
                <DiyComponentRenderer
                  component={component}
                  selected={selectedComponent === component.id}
                  previewMode={previewMode}
                  onSelect={() => onSelectComponent(component.id)}
                  onDelete={() => onDeleteComponent?.(component.id)}
                  onDuplicate={() => onDuplicateComponent?.(component.id)}
                />
              </div>
            ))}
            
            {!previewMode && (
              <DropZone
                index={components.length}
                onDrop={(dragIndex, hoverIndex) => {
                  onMoveComponent?.(dragIndex, hoverIndex);
                }}
                isLast={true}
              />
            )}
          </div>
        )}
      </div>
    </div>
  );
}
```

#### 组件渲染器
```typescript
// components/diy/DiyComponentRenderer.tsx
'use client';
import { useState } from 'react';
import { useDrag } from 'react-dnd';
import { DiyComponent } from '@/types/diy';
import { Button } from '@/components/ui/button';
import { 
  Copy, 
  Trash2, 
  Move, 
  Eye, 
  EyeOff,
  Settings 
} from 'lucide-react';

// 导入所有可用的组件
import { TextComponent } from './components/TextComponent';
import { ImageComponent } from './components/ImageComponent';
import { ButtonComponent } from './components/ButtonComponent';
import { CarouselComponent } from './components/CarouselComponent';
import { ProductListComponent } from './components/ProductListComponent';
import { VideoComponent } from './components/VideoComponent';

const componentMap = {
  text: TextComponent,
  image: ImageComponent,
  button: ButtonComponent,
  carousel: CarouselComponent,
  productList: ProductListComponent,
  video: VideoComponent,
};

interface DiyComponentRendererProps {
  component: DiyComponent;
  selected: boolean;
  previewMode: boolean;
  onSelect: () => void;
  onDelete: () => void;
  onDuplicate: () => void;
}

export function DiyComponentRenderer({
  component,
  selected,
  previewMode,
  onSelect,
  onDelete,
  onDuplicate,
}: DiyComponentRendererProps) {
  const [isHovered, setIsHovered] = useState(false);
  
  const [{ isDragging }, drag] = useDrag({
    type: 'component',
    item: { id: component.id, type: component.type },
    collect: (monitor) => ({
      isDragging: monitor.isDragging(),
    }),
  });

  // 检查显示条件
  const shouldShow = checkComponentConditions(component.conditions || []);
  if (!shouldShow && !previewMode) {
    return (
      <div className="p-4 border-2 border-dashed border-gray-300 bg-gray-50 text-center text-gray-500">
        <EyeOff className="w-4 h-4 mx-auto mb-2" />
        <p className="text-sm">组件已隐藏</p>
      </div>
    );
  }

  const ComponentToRender = componentMap[component.type as keyof typeof componentMap];
  
  if (!ComponentToRender) {
    return (
      <div className="p-4 border-2 border-dashed border-red-300 bg-red-50 text-center text-red-500">
        <p className="text-sm">未知组件类型: {component.type}</p>
      </div>
    );
  }

  // 生成CSS样式
  const componentStyle = generateStyleFromConfig(component.style);

  return (
    <div
      ref={previewMode ? undefined : drag}
      className={`
        relative group
        ${!previewMode ? 'cursor-pointer' : ''}
        ${selected ? 'ring-2 ring-blue-500' : ''}
        ${isDragging ? 'opacity-50' : ''}
      `}
      style={componentStyle}
      onClick={previewMode ? undefined : onSelect}
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    >
      {/* 组件内容 */}
      <ComponentToRender {...component.props} />
      
      {/* 编辑模式下的操作工具栏 */}
      {!previewMode && (selected || isHovered) && (
        <div className="absolute top-0 right-0 -mt-2 -mr-2 flex space-x-1 bg-white rounded-lg shadow-lg border p-1 z-10">
          <Button
            size="sm"
            variant="ghost"
            onClick={(e) => {
              e.stopPropagation();
              onDuplicate();
            }}
            className="h-6 w-6 p-0"
          >
            <Copy className="w-3 h-3" />
          </Button>
          
          <Button
            size="sm"
            variant="ghost"
            onClick={(e) => {
              e.stopPropagation();
              onDelete();
            }}
            className="h-6 w-6 p-0 text-red-500 hover:text-red-700"
          >
            <Trash2 className="w-3 h-3" />
          </Button>
          
          <div className="w-4 h-6 flex items-center justify-center cursor-move text-gray-400">
            <Move className="w-3 h-3" />
          </div>
        </div>
      )}
      
      {/* 组件标签 */}
      {!previewMode && selected && (
        <div className="absolute top-0 left-0 -mt-6 bg-blue-500 text-white text-xs px-2 py-1 rounded-t">
          {component.name}
        </div>
      )}
    </div>
  );
}

// 生成CSS样式
function generateStyleFromConfig(style: any): React.CSSProperties {
  const cssStyle: React.CSSProperties = {};
  
  // 尺寸
  if (style.width) cssStyle.width = style.width;
  if (style.height) cssStyle.height = style.height;
  
  // 间距
  if (style.margin) {
    cssStyle.marginTop = style.margin.top || 0;
    cssStyle.marginRight = style.margin.right || 0;
    cssStyle.marginBottom = style.margin.bottom || 0;
    cssStyle.marginLeft = style.margin.left || 0;
  }
  
  if (style.padding) {
    cssStyle.paddingTop = style.padding.top || 0;
    cssStyle.paddingRight = style.padding.right || 0;
    cssStyle.paddingBottom = style.padding.bottom || 0;
    cssStyle.paddingLeft = style.padding.left || 0;
  }
  
  // 边框
  if (style.border) {
    cssStyle.borderWidth = style.border.width || 0;
    cssStyle.borderStyle = style.border.style || 'none';
    cssStyle.borderColor = style.border.color || 'transparent';
  }
  
  if (style.borderRadius) {
    cssStyle.borderRadius = style.borderRadius;
  }
  
  // 背景
  if (style.background) {
    switch (style.background.type) {
      case 'color':
        cssStyle.backgroundColor = style.background.color;
        break;
      case 'image':
        cssStyle.backgroundImage = `url(${style.background.image})`;
        cssStyle.backgroundSize = style.background.size || 'cover';
        cssStyle.backgroundPosition = style.background.position || 'center';
        cssStyle.backgroundRepeat = style.background.repeat || 'no-repeat';
        break;
      case 'gradient':
        // 实现渐变背景
        if (style.background.gradient) {
          const { type, colors, direction } = style.background.gradient;
          const colorStops = colors.map((c: any) => `${c.color} ${c.position}%`).join(', ');
          if (type === 'linear') {
            cssStyle.backgroundImage = `linear-gradient(${direction || 0}deg, ${colorStops})`;
          } else {
            cssStyle.backgroundImage = `radial-gradient(circle, ${colorStops})`;
          }
        }
        break;
    }
  }
  
  // 文字样式
  if (style.fontSize) cssStyle.fontSize = style.fontSize;
  if (style.fontWeight) cssStyle.fontWeight = style.fontWeight;
  if (style.color) cssStyle.color = style.color;
  if (style.textAlign) cssStyle.textAlign = style.textAlign as any;
  
  // 阴影
  if (style.boxShadow) cssStyle.boxShadow = style.boxShadow;
  
  // 变换
  if (style.transform) cssStyle.transform = style.transform;
  
  // 层级
  if (style.zIndex) cssStyle.zIndex = style.zIndex;
  
  // 定位
  if (style.position) cssStyle.position = style.position as any;
  
  return cssStyle;
}

// 检查组件显示条件
function checkComponentConditions(conditions: any[]): boolean {
  if (conditions.length === 0) return true;
  
  return conditions.every(condition => {
    switch (condition.type) {
      case 'device':
        // 检查设备类型
        const isMobile = window.innerWidth <= 768;
        return condition.value === (isMobile ? 'mobile' : 'desktop');
        
      case 'user':
        // 检查用户状态（需要从上下文获取用户信息）
        return true; // 简化实现
        
      case 'time':
        // 检查时间条件
        const now = new Date();
        const targetTime = new Date(condition.value);
        return condition.operator === 'gt' ? now > targetTime : now < targetTime;
        
      default:
        return true;
    }
  });
}
```

---

继续写入文档...