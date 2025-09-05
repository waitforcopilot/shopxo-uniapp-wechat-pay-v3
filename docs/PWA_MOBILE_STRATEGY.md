# ShopXO PWA和移动端策略详解

## 第六部分：PWA和移动端优化

### 6.1 PWA配置和Service Worker实现

#### PWA清单文件
```json
// public/manifest.json
{
  "name": "ShopXO 商城 - 专业跨境电商平台",
  "short_name": "ShopXO",
  "description": "韩国香港免税店直采，正品美妆护肤品跨境销售平台",
  "lang": "zh-CN",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#ff0036",
  "background_color": "#ffffff",
  "orientation": "portrait-primary",
  "scope": "/",
  "id": "shopxo-ecommerce-app",
  "categories": ["shopping", "lifestyle", "business"],
  "screenshots": [
    {
      "src": "/screenshots/home-mobile.png",
      "sizes": "390x844",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "首页展示"
    },
    {
      "src": "/screenshots/product-mobile.png",
      "sizes": "390x844", 
      "type": "image/png",
      "form_factor": "narrow",
      "label": "商品详情"
    },
    {
      "src": "/screenshots/cart-mobile.png",
      "sizes": "390x844",
      "type": "image/png", 
      "form_factor": "narrow",
      "label": "购物车"
    },
    {
      "src": "/screenshots/home-desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide",
      "label": "桌面端首页"
    }
  ],
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-96x96.png", 
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128", 
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-192x192.png", 
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png", 
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "shortcuts": [
    {
      "name": "购物车",
      "short_name": "购物车", 
      "description": "查看购物车中的商品",
      "url": "/cart",
      "icons": [
        {
          "src": "/icons/cart-96x96.png",
          "sizes": "96x96",
          "type": "image/png"
        }
      ]
    },
    {
      "name": "我的订单",
      "short_name": "订单",
      "description": "查看我的订单状态",
      "url": "/user/orders", 
      "icons": [
        {
          "src": "/icons/orders-96x96.png",
          "sizes": "96x96", 
          "type": "image/png"
        }
      ]
    },
    {
      "name": "收藏夹",
      "short_name": "收藏",
      "description": "查看收藏的商品",
      "url": "/user/favorites",
      "icons": [
        {
          "src": "/icons/heart-96x96.png", 
          "sizes": "96x96",
          "type": "image/png"
        }
      ]
    },
    {
      "name": "优惠活动",
      "short_name": "优惠",
      "description": "查看最新优惠活动",
      "url": "/activities",
      "icons": [
        {
          "src": "/icons/gift-96x96.png",
          "sizes": "96x96",
          "type": "image/png"
        }
      ]
    }
  ],
  "related_applications": [
    {
      "platform": "play",
      "url": "https://play.google.com/store/apps/details?id=com.shopxo.app",
      "id": "com.shopxo.app"
    },
    {
      "platform": "itunes",
      "url": "https://apps.apple.com/app/shopxo/id123456789"
    }
  ],
  "prefer_related_applications": false,
  "edge_side_panel": {
    "preferred_width": 400
  },
  "display_override": ["window-controls-overlay", "standalone", "minimal-ui"],
  "protocol_handlers": [
    {
      "protocol": "web+shopxo",
      "url": "/share?url=%s"
    }
  ]
}
```

#### 高级Service Worker实现
```javascript
// public/sw.js
const CACHE_NAME = 'shopxo-v1.2.0';
const STATIC_CACHE = `${CACHE_NAME}-static`;
const DYNAMIC_CACHE = `${CACHE_NAME}-dynamic`;
const IMAGES_CACHE = `${CACHE_NAME}-images`;
const API_CACHE = `${CACHE_NAME}-api`;

// 缓存策略配置
const CACHE_STRATEGIES = {
  // 静态资源：缓存优先（Cache First）
  static: [
    '/',
    '/offline',
    '/manifest.json',
    '/_next/static/',
    '/icons/',
    '/images/common/',
  ],
  
  // API请求：网络优先，离线回退（Network First）
  api: [
    '/api/products',
    '/api/categories', 
    '/api/user',
    '/api/cart',
  ],
  
  // 图片：缓存优先，过期重新验证（Stale While Revalidate）
  images: [
    'https://nsr.itmacode.cn/',
    'https://cdn.shopxo.net/',
  ],
  
  // 页面：网络优先，离线页面回退（Network First with Offline Fallback）
  pages: [
    '/products/',
    '/category/',
    '/user/',
  ]
};

// 预缓存的关键资源
const PRECACHE_RESOURCES = [
  '/',
  '/offline',
  '/manifest.json',
  '/icons/icon-192x192.png',
  '/icons/icon-512x512.png',
  // 关键CSS和JS会由Workbox自动处理
];

// 安装事件 - 预缓存关键资源
self.addEventListener('install', (event) => {
  console.log('Service Worker installing...');
  
  event.waitUntil(
    Promise.all([
      // 预缓存关键资源
      caches.open(STATIC_CACHE).then((cache) => {
        console.log('Pre-caching static resources');
        return cache.addAll(PRECACHE_RESOURCES);
      }),
      
      // 跳过等待，立即激活
      self.skipWaiting(),
    ])
  );
});

// 激活事件 - 清理旧缓存
self.addEventListener('activate', (event) => {
  console.log('Service Worker activating...');
  
  event.waitUntil(
    Promise.all([
      // 清理旧版本缓存
      caches.keys().then((cacheNames) => {
        return Promise.all(
          cacheNames
            .filter(cacheName => !cacheName.startsWith(CACHE_NAME))
            .map(cacheName => {
              console.log('Deleting old cache:', cacheName);
              return caches.delete(cacheName);
            })
        );
      }),
      
      // 立即控制所有页面
      self.clients.claim(),
      
      // 预热重要缓存
      warmupCache(),
    ])
  );
});

// 网络请求拦截
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // 跳过非GET请求
  if (request.method !== 'GET') {
    return;
  }

  // 跳过浏览器扩展请求
  if (url.protocol === 'chrome-extension:' || url.protocol === 'moz-extension:') {
    return;
  }

  // API请求：网络优先策略
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirstStrategy(request, API_CACHE));
    return;
  }

  // 图片请求：过期重新验证策略
  if (request.destination === 'image') {
    event.respondWith(staleWhileRevalidateStrategy(request, IMAGES_CACHE));
    return;
  }

  // 静态资源：缓存优先策略
  if (isStaticResource(url.pathname)) {
    event.respondWith(cacheFirstStrategy(request, STATIC_CACHE));
    return;
  }

  // 页面请求：网络优先，离线页面回退
  if (request.mode === 'navigate') {
    event.respondWith(navigationStrategy(request));
    return;
  }

  // 其他请求：网络优先
  event.respondWith(networkFirstStrategy(request, DYNAMIC_CACHE));
});

// 缓存优先策略 (Cache First)
async function cacheFirstStrategy(request, cacheName = STATIC_CACHE) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);
  
  if (cached) {
    console.log('Cache hit:', request.url);
    return cached;
  }

  console.log('Cache miss, fetching:', request.url);
  try {
    const response = await fetch(request);
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  } catch (error) {
    console.error('Fetch failed:', error);
    
    // 如果是图片请求失败，返回占位图
    if (request.destination === 'image') {
      return caches.match('/icons/image-placeholder.png');
    }
    
    throw error;
  }
}

// 网络优先策略 (Network First)
async function networkFirstStrategy(request, cacheName = DYNAMIC_CACHE) {
  const cache = await caches.open(cacheName);
  
  try {
    console.log('Network first, fetching:', request.url);
    const response = await fetch(request);
    
    if (response.ok) {
      // 只缓存成功的响应
      cache.put(request, response.clone());
    }
    
    return response;
  } catch (error) {
    console.log('Network failed, trying cache:', request.url);
    const cached = await cache.match(request);
    
    if (cached) {
      console.log('Cache hit after network failure:', request.url);
      return cached;
    }
    
    console.error('Both network and cache failed:', error);
    throw error;
  }
}

// 过期重新验证策略 (Stale While Revalidate)
async function staleWhileRevalidateStrategy(request, cacheName = IMAGES_CACHE) {
  const cache = await caches.open(cacheName);
  const cached = await cache.match(request);
  
  // 立即返回缓存的响应（如果有）
  const response = cached || fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  });

  // 在后台更新缓存
  if (cached) {
    console.log('Serving from cache, updating in background:', request.url);
    fetch(request).then(fetchResponse => {
      if (fetchResponse.ok) {
        cache.put(request, fetchResponse);
      }
    }).catch(error => {
      console.log('Background update failed:', error);
    });
  }

  return response;
}

// 导航策略 (Navigation Strategy)
async function navigationStrategy(request) {
  try {
    console.log('Navigation request:', request.url);
    const response = await fetch(request);
    
    // 缓存成功的页面响应
    if (response.ok) {
      const cache = await caches.open(DYNAMIC_CACHE);
      cache.put(request, response.clone());
    }
    
    return response;
  } catch (error) {
    console.log('Navigation failed, serving offline page:', error);
    
    // 尝试从缓存获取页面
    const cache = await caches.open(DYNAMIC_CACHE);
    const cached = await cache.match(request);
    
    if (cached) {
      return cached;
    }
    
    // 返回离线页面
    return caches.match('/offline');
  }
}

// 工具函数：检查是否为静态资源
function isStaticResource(pathname) {
  return CACHE_STRATEGIES.static.some(pattern => 
    pathname.startsWith(pattern)
  );
}

// 预热重要缓存
async function warmupCache() {
  try {
    const cache = await caches.open(DYNAMIC_CACHE);
    
    // 预热关键API
    const warmupRequests = [
      '/api/categories',
      '/api/products?hot=true&limit=10',
      '/api/diy/home',
    ];
    
    await Promise.allSettled(
      warmupRequests.map(url => 
        fetch(url).then(response => {
          if (response.ok) {
            cache.put(url, response);
          }
        })
      )
    );
    
    console.log('Cache warmup completed');
  } catch (error) {
    console.error('Cache warmup failed:', error);
  }
}

// 后台同步事件处理
self.addEventListener('sync', (event) => {
  console.log('Background sync triggered:', event.tag);
  
  if (event.tag === 'cart-sync') {
    event.waitUntil(syncCartData());
  }
  
  if (event.tag === 'order-sync') {
    event.waitUntil(syncOrderData());
  }
  
  if (event.tag === 'analytics-sync') {
    event.waitUntil(syncAnalyticsData());
  }
});

// 购物车数据同步
async function syncCartData() {
  try {
    const pendingCarts = await getFromIndexedDB('pending-carts');
    
    for (const cartData of pendingCarts) {
      const response = await fetch('/api/cart/sync', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(cartData),
      });
      
      if (response.ok) {
        await removeFromIndexedDB('pending-carts', cartData.id);
        console.log('Cart data synced successfully');
      }
    }
  } catch (error) {
    console.error('Cart sync failed:', error);
  }
}

// 订单数据同步
async function syncOrderData() {
  try {
    const pendingOrders = await getFromIndexedDB('pending-orders');
    
    for (const orderData of pendingOrders) {
      const response = await fetch('/api/orders/sync', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(orderData),
      });
      
      if (response.ok) {
        await removeFromIndexedDB('pending-orders', orderData.id);
        console.log('Order data synced successfully');
      }
    }
  } catch (error) {
    console.error('Order sync failed:', error);
  }
}

// 分析数据同步
async function syncAnalyticsData() {
  try {
    const pendingEvents = await getFromIndexedDB('pending-analytics');
    
    if (pendingEvents.length > 0) {
      const response = await fetch('/api/analytics/batch', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ events: pendingEvents }),
      });
      
      if (response.ok) {
        await clearIndexedDBStore('pending-analytics');
        console.log('Analytics data synced successfully');
      }
    }
  } catch (error) {
    console.error('Analytics sync failed:', error);
  }
}

// 推送通知处理
self.addEventListener('push', (event) => {
  if (!event.data) return;

  const data = event.data.json();
  console.log('Push notification received:', data);

  const options = {
    body: data.body,
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    tag: data.tag || 'default',
    data: data.data,
    actions: data.actions || [
      {
        action: 'view',
        title: '查看详情',
        icon: '/icons/view-16x16.png'
      },
      {
        action: 'dismiss',
        title: '忽略',
        icon: '/icons/close-16x16.png'
      }
    ],
    requireInteraction: data.urgent || false,
    silent: data.silent || false,
    vibrate: data.vibrate || [200, 100, 200],
    timestamp: Date.now(),
  };

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// 通知点击处理
self.addEventListener('notificationclick', (event) => {
  console.log('Notification clicked:', event.notification.data);
  
  event.notification.close();

  const action = event.action;
  const data = event.notification.data;
  
  if (action === 'dismiss') {
    return; // 用户选择忽略
  }

  // 处理通知点击
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then((clientList) => {
      // 如果已有窗口打开，聚焦到该窗口
      for (const client of clientList) {
        if (client.url === data.url && 'focus' in client) {
          return client.focus();
        }
      }
      
      // 否则打开新窗口
      if (clients.openWindow) {
        return clients.openWindow(data.url || '/');
      }
    })
  );
});

// IndexedDB 操作工具函数
async function getFromIndexedDB(storeName) {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('ShopXODB', 1);
    
    request.onerror = () => reject(request.error);
    request.onsuccess = () => {
      const db = request.result;
      const transaction = db.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const getRequest = store.getAll();
      
      getRequest.onerror = () => reject(getRequest.error);
      getRequest.onsuccess = () => resolve(getRequest.result);
    };
    
    request.onupgradeneeded = () => {
      const db = request.result;
      if (!db.objectStoreNames.contains(storeName)) {
        db.createObjectStore(storeName, { keyPath: 'id', autoIncrement: true });
      }
    };
  });
}

async function removeFromIndexedDB(storeName, id) {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('ShopXODB', 1);
    
    request.onerror = () => reject(request.error);
    request.onsuccess = () => {
      const db = request.result;
      const transaction = db.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const deleteRequest = store.delete(id);
      
      deleteRequest.onerror = () => reject(deleteRequest.error);
      deleteRequest.onsuccess = () => resolve();
    };
  });
}

async function clearIndexedDBStore(storeName) {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('ShopXODB', 1);
    
    request.onerror = () => reject(request.error);
    request.onsuccess = () => {
      const db = request.result;
      const transaction = db.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const clearRequest = store.clear();
      
      clearRequest.onerror = () => reject(clearRequest.error);
      clearRequest.onsuccess = () => resolve();
    };
  });
}

// 错误处理
self.addEventListener('error', (event) => {
  console.error('Service Worker error:', event.error);
});

self.addEventListener('unhandledrejection', (event) => {
  console.error('Service Worker unhandled rejection:', event.reason);
});

console.log('Service Worker loaded successfully');
```

### 6.2 响应式设计系统

#### Tailwind CSS 配置优化
```javascript
// tailwind.config.js
const { fontFamily } = require('tailwindcss/defaultTheme');

/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ['class'],
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    container: {
      center: true,
      padding: '2rem',
      screens: {
        '2xl': '1400px',
      },
    },
    screens: {
      'xs': '375px',      // 小屏手机
      'sm': '640px',      // 大屏手机
      'md': '768px',      // 平板
      'lg': '1024px',     // 小屏笔记本
      'xl': '1280px',     // 大屏笔记本
      '2xl': '1536px',    // 桌面显示器
      
      // 自定义断点
      'mobile': { 'max': '767px' },
      'tablet': { 'min': '768px', 'max': '1023px' },
      'desktop': { 'min': '1024px' },
      
      // 设备方向
      'portrait': { 'raw': '(orientation: portrait)' },
      'landscape': { 'raw': '(orientation: landscape)' },
      
      // 高分辨率屏幕
      'retina': { 'raw': '(-webkit-min-device-pixel-ratio: 2)' },
    },
    extend: {
      // 安全区域适配
      spacing: {
        'safe-top': 'env(safe-area-inset-top)',
        'safe-bottom': 'env(safe-area-inset-bottom)',
        'safe-left': 'env(safe-area-inset-left)',
        'safe-right': 'env(safe-area-inset-right)',
        
        // 触摸友好的尺寸
        'touch': '44px', // iOS HIG推荐的最小触摸目标
        'touch-lg': '48px', // Material Design推荐
      },
      
      // 动态视窗单位
      minHeight: {
        'dvh': '100dvh',    // 动态视窗高度
        'svh': '100svh',    // 小视窗高度
        'lvh': '100lvh',    // 大视窗高度
        'touch': '44px',
      },
      
      height: {
        'dvh': '100dvh',
        'svh': '100svh', 
        'lvh': '100lvh',
      },
      
      // 主题颜色变量
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
          50: 'hsl(var(--primary-50))',
          100: 'hsl(var(--primary-100))',
          200: 'hsl(var(--primary-200))',
          300: 'hsl(var(--primary-300))',
          400: 'hsl(var(--primary-400))',
          500: 'hsl(var(--primary-500))',
          600: 'hsl(var(--primary-600))',
          700: 'hsl(var(--primary-700))',
          800: 'hsl(var(--primary-800))',
          900: 'hsl(var(--primary-900))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
      },
      
      // 字体配置
      fontFamily: {
        sans: ['var(--font-sans)', ...fontFamily.sans],
        mono: ['var(--font-mono)', ...fontFamily.mono],
        
        // 中文字体优化
        'chinese': [
          'PingFang SC',
          'Hiragino Sans GB',
          'Microsoft YaHei',
          'WenQuanYi Micro Hei',
          'sans-serif'
        ],
      },
      
      // 边框圆角
      borderRadius: {
        'xs': '0.125rem',
        'lg': 'var(--radius)',
        'md': 'calc(var(--radius) - 2px)',
        'sm': 'calc(var(--radius) - 4px)',
      },
      
      // 动画和过渡
      keyframes: {
        'accordion-down': {
          from: { height: 0 },
          to: { height: 'var(--radix-accordion-content-height)' },
        },
        'accordion-up': {
          from: { height: 'var(--radix-accordion-content-height)' },
          to: { height: 0 },
        },
        'slide-in-from-bottom': {
          '0%': { transform: 'translateY(100%)' },
          '100%': { transform: 'translateY(0)' },
        },
        'slide-in-from-top': {
          '0%': { transform: 'translateY(-100%)' },
          '100%': { transform: 'translateY(0)' },
        },
        'fade-in': {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        'scale-in': {
          '0%': { transform: 'scale(0.95)', opacity: '0' },
          '100%': { transform: 'scale(1)', opacity: '1' },
        },
        'shimmer': {
          '0%': { backgroundPosition: '-200% 0' },
          '100%': { backgroundPosition: '200% 0' },
        },
      },
      animation: {
        'accordion-down': 'accordion-down 0.2s ease-out',
        'accordion-up': 'accordion-up 0.2s ease-out',
        'slide-in-from-bottom': 'slide-in-from-bottom 0.3s ease-out',
        'slide-in-from-top': 'slide-in-from-top 0.3s ease-out',
        'fade-in': 'fade-in 0.2s ease-out',
        'scale-in': 'scale-in 0.2s ease-out',
        'shimmer': 'shimmer 2s linear infinite',
      },
      
      // 自定义工具类
      backgroundImage: {
        'shimmer': 'linear-gradient(90deg, transparent, rgba(255,255,255,0.2), transparent)',
      },
      
      // 阴影
      boxShadow: {
        'soft': '0 2px 8px 0 rgba(0, 0, 0, 0.05)',
        'medium': '0 4px 16px 0 rgba(0, 0, 0, 0.1)',
        'hard': '0 8px 32px 0 rgba(0, 0, 0, 0.15)',
      },
      
      // Z-index层级
      zIndex: {
        'negative': '-1',
        'modal': '1000',
        'popover': '1010',
        'overlay': '1020',
        'dropdown': '1030',
        'tooltip': '1040',
        'notification': '1050',
      },
    },
  },
  plugins: [
    require('tailwindcss-animate'),
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('tailwindcss-safe-area'),
    
    // 自定义插件
    function({ addUtilities, theme }) {
      // 添加触摸友好的工具类
      addUtilities({
        '.touch-manipulation': {
          'touch-action': 'manipulation',
        },
        '.scroll-smooth': {
          'scroll-behavior': 'smooth',
        },
        '.scroll-touch': {
          '-webkit-overflow-scrolling': 'touch',
        },
        
        // 安全区域工具类
        '.pb-safe': {
          'padding-bottom': 'env(safe-area-inset-bottom)',
        },
        '.pt-safe': {
          'padding-top': 'env(safe-area-inset-top)',
        },
        '.pl-safe': {
          'padding-left': 'env(safe-area-inset-left)',
        },
        '.pr-safe': {
          'padding-right': 'env(safe-area-inset-right)',
        },
        
        // 1px边框解决方案
        '.border-hairline': {
          'position': 'relative',
          '&::after': {
            'content': '""',
            'position': 'absolute',
            'top': '0',
            'left': '0',
            'width': '200%',
            'height': '200%',
            'border': '1px solid currentColor',
            'transform': 'scale(0.5)',
            'transform-origin': 'top left',
            'pointer-events': 'none',
          },
        },
        
        // 文字截断
        '.text-ellipsis-2': {
          'display': '-webkit-box',
          '-webkit-line-clamp': '2',
          '-webkit-box-orient': 'vertical',
          'overflow': 'hidden',
        },
        '.text-ellipsis-3': {
          'display': '-webkit-box',
          '-webkit-line-clamp': '3',
          '-webkit-box-orient': 'vertical',
          'overflow': 'hidden',
        },
      });
    },
  ],
};
```

#### 移动端布局组件
```typescript
// components/layout/MobileLayout.tsx
'use client';
import { useEffect, useState, useRef } from 'react';
import { cn } from '@/lib/utils';
import { MobileTabBar } from './MobileTabBar';
import { MobileHeader } from './MobileHeader';
import { useRouter, usePathname } from 'next/navigation';
import { useAuth } from '@/hooks/use-auth';
import { useCartStore } from '@/store/cart';

interface MobileLayoutProps {
  children: React.ReactNode;
  showTabBar?: boolean;
  showHeader?: boolean;
  headerProps?: {
    title?: string;
    showBackButton?: boolean;
    showSearchButton?: boolean;
    showCartButton?: boolean;
    transparent?: boolean;
    fixed?: boolean;
  };
  fullscreen?: boolean;
  className?: string;
}

export function MobileLayout({
  children,
  showTabBar = true,
  showHeader = true,
  headerProps = {},
  fullscreen = false,
  className,
}: MobileLayoutProps) {
  const [isScrollingUp, setIsScrollingUp] = useState(true);
  const [lastScrollY, setLastScrollY] = useState(0);
  const [headerHeight, setHeaderHeight] = useState(0);
  const [tabBarHeight, setTabBarHeight] = useState(0);
  
  const router = useRouter();
  const pathname = usePathname();
  const { isAuthenticated } = useAuth();
  const cartItems = useCartStore(state => state.items);
  
  const headerRef = useRef<HTMLDivElement>(null);
  const tabBarRef = useRef<HTMLDivElement>(null);
  const contentRef = useRef<HTMLDivElement>(null);

  // 滚动方向检测
  useEffect(() => {
    const handleScroll = () => {
      const currentScrollY = window.scrollY;
      
      // 向上滚动或滚动距离很小时显示导航栏
      if (currentScrollY < lastScrollY || currentScrollY < 50) {
        setIsScrollingUp(true);
      } 
      // 向下滚动且距离较大时隐藏导航栏
      else if (currentScrollY > lastScrollY && currentScrollY > 100) {
        setIsScrollingUp(false);
      }
      
      setLastScrollY(currentScrollY);
    };

    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, [lastScrollY]);

  // 计算布局高度
  useEffect(() => {
    if (headerRef.current) {
      setHeaderHeight(headerRef.current.offsetHeight);
    }
    if (tabBarRef.current) {
      setTabBarHeight(tabBarRef.current.offsetHeight);
    }
  }, [showHeader, showTabBar]);

  // 处理返回按钮
  const handleBack = () => {
    if (window.history.length > 1) {
      router.back();
    } else {
      router.push('/');
    }
  };

  // 处理搜索按钮
  const handleSearch = () => {
    router.push('/search');
  };

  // 处理购物车按钮
  const handleCart = () => {
    if (!isAuthenticated) {
      router.push('/login?redirect=/cart');
    } else {
      router.push('/cart');
    }
  };

  return (
    <div className={cn(
      "flex flex-col min-h-dvh bg-gray-50",
      "touch-manipulation scroll-smooth",
      fullscreen && "h-dvh overflow-hidden",
      className
    )}>
      {/* 状态栏占位 */}
      <div className="h-safe-top bg-white" />
      
      {/* 顶部导航栏 */}
      {showHeader && (
        <MobileHeader
          ref={headerRef}
          title={headerProps.title}
          showBackButton={headerProps.showBackButton}
          showSearchButton={headerProps.showSearchButton}
          showCartButton={headerProps.showCartButton}
          transparent={headerProps.transparent}
          fixed={headerProps.fixed}
          cartItemCount={cartItems.length}
          onBack={handleBack}
          onSearch={handleSearch}
          onCart={handleCart}
          className={cn(
            "transition-transform duration-300 ease-in-out",
            !isScrollingUp && headerProps.fixed && "transform -translate-y-full"
          )}
        />
      )}

      {/* 主内容区域 */}
      <main 
        ref={contentRef}
        className={cn(
          "flex-1 relative",
          showTabBar && "pb-16 pb-safe", // 为底部导航栏留出空间
          showHeader && headerProps.fixed && "pt-0", // 固定头部时不需要额外padding
          fullscreen ? "overflow-hidden" : "overflow-visible"
        )}
        style={{
          paddingTop: showHeader && headerProps.fixed ? headerHeight : 0,
        }}
      >
        {children}
      </main>

      {/* 底部导航栏 */}
      {showTabBar && (
        <MobileTabBar 
          ref={tabBarRef}
          currentPath={pathname}
          cartItemCount={cartItems.length}
          className={cn(
            "transition-transform duration-300 ease-in-out",
            !isScrollingUp && "transform translate-y-full"
          )}
        />
      )}
      
      {/* 底部安全区域 */}
      <div className="h-safe-bottom bg-white" />
    </div>
  );
}
```

#### 自适应导航栏
```typescript
// components/layout/MobileHeader.tsx
'use client';
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { 
  ArrowLeft, 
  Search, 
  ShoppingCart, 
  Menu,
  MoreHorizontal 
} from 'lucide-react';

interface MobileHeaderProps {
  title?: string;
  showBackButton?: boolean;
  showSearchButton?: boolean;
  showCartButton?: boolean;
  showMenuButton?: boolean;
  transparent?: boolean;
  fixed?: boolean;
  cartItemCount?: number;
  onBack?: () => void;
  onSearch?: () => void;
  onCart?: () => void;
  onMenu?: () => void;
  leftContent?: React.ReactNode;
  rightContent?: React.ReactNode;
  className?: string;
}

export const MobileHeader = forwardRef<HTMLDivElement, MobileHeaderProps>(({
  title,
  showBackButton = false,
  showSearchButton = false,
  showCartButton = false,
  showMenuButton = false,
  transparent = false,
  fixed = true,
  cartItemCount = 0,
  onBack,
  onSearch,
  onCart,
  onMenu,
  leftContent,
  rightContent,
  className,
}, ref) => {
  return (
    <header 
      ref={ref}
      className={cn(
        "flex items-center justify-between px-4 h-14",
        "border-b border-gray-200 z-50",
        transparent 
          ? "bg-transparent border-transparent" 
          : "bg-white/95 backdrop-blur-sm",
        fixed && "sticky top-0",
        className
      )}
    >
      {/* 左侧内容 */}
      <div className="flex items-center space-x-2 flex-1">
        {leftContent || (
          <>
            {showBackButton && (
              <Button
                variant="ghost"
                size="sm"
                onClick={onBack}
                className="p-2 h-auto"
                aria-label="返回"
              >
                <ArrowLeft className="h-5 w-5" />
              </Button>
            )}
            
            {showMenuButton && (
              <Button
                variant="ghost"
                size="sm"
                onClick={onMenu}
                className="p-2 h-auto"
                aria-label="菜单"
              >
                <Menu className="h-5 w-5" />
              </Button>
            )}
          </>
        )}
      </div>

      {/* 中间标题 */}
      {title && (
        <div className="flex-1 text-center">
          <h1 className="text-lg font-semibold text-gray-900 truncate px-4">
            {title}
          </h1>
        </div>
      )}

      {/* 右侧内容 */}
      <div className="flex items-center space-x-2 flex-1 justify-end">
        {rightContent || (
          <>
            {showSearchButton && (
              <Button
                variant="ghost"
                size="sm"
                onClick={onSearch}
                className="p-2 h-auto"
                aria-label="搜索"
              >
                <Search className="h-5 w-5" />
              </Button>
            )}
            
            {showCartButton && (
              <Button
                variant="ghost"
                size="sm"
                onClick={onCart}
                className="p-2 h-auto relative"
                aria-label={`购物车${cartItemCount > 0 ? ` (${cartItemCount}件商品)` : ''}`}
              >
                <ShoppingCart className="h-5 w-5" />
                {cartItemCount > 0 && (
                  <Badge 
                    variant="destructive" 
                    className="absolute -top-1 -right-1 min-w-[18px] h-[18px] p-0 text-xs flex items-center justify-center"
                  >
                    {cartItemCount > 99 ? '99+' : cartItemCount}
                  </Badge>
                )}
              </Button>
            )}
          </>
        )}
      </div>
    </header>
  );
});

MobileHeader.displayName = 'MobileHeader';
```

#### 底部标签栏
```typescript
// components/layout/MobileTabBar.tsx
'use client';
import { forwardRef } from 'react';
import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { cn } from '@/lib/utils';
import { Badge } from '@/components/ui/badge';
import { 
  Home, 
  Grid3X3, 
  ShoppingCart, 
  User,
  Heart,
  Search
} from 'lucide-react';

interface TabBarItem {
  key: string;
  label: string;
  icon: React.ComponentType<{ className?: string }>;
  path: string;
  activePattern: RegExp;
  showBadge?: boolean;
  badgeCount?: number;
}

interface MobileTabBarProps {
  currentPath?: string;
  cartItemCount?: number;
  favoriteCount?: number;
  className?: string;
}

export const MobileTabBar = forwardRef<HTMLDivElement, MobileTabBarProps>(({
  currentPath,
  cartItemCount = 0,
  favoriteCount = 0,
  className,
}, ref) => {
  const pathname = usePathname();
  const currentRoute = currentPath || pathname;

  const tabBarItems: TabBarItem[] = [
    {
      key: 'home',
      label: '首页',
      icon: Home,
      path: '/',
      activePattern: /^\/$/,
    },
    {
      key: 'category',
      label: '分类',
      icon: Grid3X3,
      path: '/category',
      activePattern: /^\/category/,
    },
    {
      key: 'cart',
      label: '购物车',
      icon: ShoppingCart,
      path: '/cart',
      activePattern: /^\/cart/,
      showBadge: true,
      badgeCount: cartItemCount,
    },
    {
      key: 'favorites',
      label: '收藏',
      icon: Heart,
      path: '/user/favorites',
      activePattern: /^\/user\/favorites/,
      showBadge: favoriteCount > 0,
      badgeCount: favoriteCount,
    },
    {
      key: 'user',
      label: '我的',
      icon: User,
      path: '/user',
      activePattern: /^\/user(?!\/favorites)/,
    },
  ];

  return (
    <nav 
      ref={ref}
      className={cn(
        "fixed bottom-0 left-0 right-0",
        "bg-white/95 backdrop-blur-sm border-t border-gray-200",
        "pb-safe z-50",
        className
      )}
    >
      <div className="flex items-center justify-around h-16 px-2">
        {tabBarItems.map((item) => {
          const Icon = item.icon;
          const isActive = item.activePattern.test(currentRoute);
          
          return (
            <Link
              key={item.key}
              href={item.path}
              className={cn(
                "flex flex-col items-center justify-center space-y-1",
                "relative px-3 py-2 rounded-lg transition-all duration-200",
                "min-h-touch touch-manipulation",
                "active:scale-95",
                isActive 
                  ? "text-primary-600 bg-primary-50" 
                  : "text-gray-500 hover:text-gray-700 active:bg-gray-100"
              )}
              aria-label={item.label}
            >
              <div className="relative">
                <Icon className={cn(
                  "w-6 h-6 transition-all duration-200",
                  isActive && "scale-110"
                )} />
                
                {/* 徽章显示 */}
                {item.showBadge && item.badgeCount && item.badgeCount > 0 && (
                  <Badge 
                    variant="destructive" 
                    className={cn(
                      "absolute -top-2 -right-2",
                      "min-w-[18px] h-[18px] p-0 text-xs",
                      "flex items-center justify-center",
                      "animate-scale-in"
                    )}
                  >
                    {item.badgeCount > 99 ? '99+' : item.badgeCount}
                  </Badge>
                )}
                
                {/* 活跃状态指示器 */}
                {isActive && (
                  <div className="absolute -bottom-1 left-1/2 transform -translate-x-1/2 w-1 h-1 bg-primary-600 rounded-full" />
                )}
              </div>
              
              <span className={cn(
                "text-xs font-medium transition-colors duration-200",
                isActive ? "text-primary-600" : "text-gray-500"
              )}>
                {item.label}
              </span>
            </Link>
          );
        })}
      </div>
    </nav>
  );
});

MobileTabBar.displayName = 'MobileTabBar';
```

### 6.3 手势和交互优化

#### 手势处理Hook
```typescript
// hooks/use-touch-gestures.ts
import { useCallback, useRef, useState } from 'react';

interface TouchPoint {
  x: number;
  y: number;
  timestamp: number;
}

interface GestureConfig {
  onSwipeLeft?: () => void;
  onSwipeRight?: () => void;
  onSwipeUp?: () => void;
  onSwipeDown?: () => void;
  onPinchStart?: () => void;
  onPinch?: (scale: number, velocity: number) => void;
  onPinchEnd?: () => void;
  onLongPress?: (point: TouchPoint) => void;
  onDoubleTap?: (point: TouchPoint) => void;
  swipeThreshold?: number;
  longPressThreshold?: number;
  doubleTapThreshold?: number;
  pinchThreshold?: number;
}

export function useTouchGestures(config: GestureConfig) {
  const {
    onSwipeLeft,
    onSwipeRight,
    onSwipeUp,
    onSwipeDown,
    onPinchStart,
    onPinch,
    onPinchEnd,
    onLongPress,
    onDoubleTap,
    swipeThreshold = 50,
    longPressThreshold = 500,
    doubleTapThreshold = 300,
    pinchThreshold = 10,
  } = config;

  const touchStartRef = useRef<TouchPoint | null>(null);
  const touchEndRef = useRef<TouchPoint | null>(null);
  const lastTapRef = useRef<TouchPoint | null>(null);
  const longPressTimerRef = useRef<NodeJS.Timeout | null>(null);
  const initialDistanceRef = useRef<number>(0);
  const isPinchingRef = useRef<boolean>(false);
  const [isLongPressing, setIsLongPressing] = useState(false);

  // 计算两点距离
  const getDistance = useCallback((touch1: Touch, touch2: Touch): number => {
    const deltaX = touch1.clientX - touch2.clientX;
    const deltaY = touch1.clientY - touch2.clientY;
    return Math.sqrt(deltaX * deltaX + deltaY * deltaY);
  }, []);

  // 开始触摸
  const onTouchStart = useCallback((e: React.TouchEvent) => {
    const now = Date.now();
    
    if (e.touches.length === 1) {
      const touch = e.touches[0];
      const point: TouchPoint = {
        x: touch.clientX,
        y: touch.clientY,
        timestamp: now,
      };
      
      touchStartRef.current = point;
      touchEndRef.current = null;

      // 检测双击
      if (lastTapRef.current && onDoubleTap) {
        const timeDiff = now - lastTapRef.current.timestamp;
        const distance = Math.sqrt(
          Math.pow(point.x - lastTapRef.current.x, 2) + 
          Math.pow(point.y - lastTapRef.current.y, 2)
        );
        
        if (timeDiff < doubleTapThreshold && distance < 50) {
          onDoubleTap(point);
          lastTapRef.current = null;
          return;
        }
      }

      // 设置长按定时器
      if (onLongPress) {
        longPressTimerRef.current = setTimeout(() => {
          setIsLongPressing(true);
          onLongPress(point);
        }, longPressThreshold);
      }

    } else if (e.touches.length === 2) {
      // 双指操作
      const touch1 = e.touches[0];
      const touch2 = e.touches[1];
      initialDistanceRef.current = getDistance(touch1, touch2);
      isPinchingRef.current = false;
      
      // 清除长按定时器
      if (longPressTimerRef.current) {
        clearTimeout(longPressTimerRef.current);
        longPressTimerRef.current = null;
      }
      
      if (onPinchStart) {
        onPinchStart();
      }
    }
  }, [getDistance, onDoubleTap, onLongPress, onPinchStart, doubleTapThreshold, longPressThreshold]);

  // 触摸移动
  const onTouchMove = useCallback((e: React.TouchEvent) => {
    // 清除长按定时器
    if (longPressTimerRef.current) {
      clearTimeout(longPressTimerRef.current);
      longPressTimerRef.current = null;
      setIsLongPressing(false);
    }

    if (e.touches.length === 2 && onPinch) {
      const touch1 = e.touches[0];
      const touch2 = e.touches[1];
      const currentDistance = getDistance(touch1, touch2);
      
      if (initialDistanceRef.current > 0) {
        const scale = currentDistance / initialDistanceRef.current;
        const distanceDiff = Math.abs(currentDistance - initialDistanceRef.current);
        
        if (distanceDiff > pinchThreshold || isPinchingRef.current) {
          isPinchingRef.current = true;
          
          // 计算缩放速度
          const velocity = distanceDiff / initialDistanceRef.current;
          onPinch(scale, velocity);
        }
      }
    }
  }, [getDistance, onPinch, pinchThreshold]);

  // 触摸结束
  const onTouchEnd = useCallback((e: React.TouchEvent) => {
    const now = Date.now();
    
    // 清除长按定时器
    if (longPressTimerRef.current) {
      clearTimeout(longPressTimerRef.current);
      longPressTimerRef.current = null;
    }
    
    setIsLongPressing(false);

    if (e.touches.length === 0) {
      // 所有手指都离开了屏幕
      if (isPinchingRef.current && onPinchEnd) {
        onPinchEnd();
        isPinchingRef.current = false;
        return;
      }

      // 单指滑动检测
      if (!touchStartRef.current || e.changedTouches.length === 0) return;
      
      const touch = e.changedTouches[0];
      touchEndRef.current = {
        x: touch.clientX,
        y: touch.clientY,
        timestamp: now,
      };

      const deltaX = touchEndRef.current.x - touchStartRef.current.x;
      const deltaY = touchEndRef.current.y - touchStartRef.current.y;
      const absX = Math.abs(deltaX);
      const absY = Math.abs(deltaY);
      
      // 判断是否为有效滑动
      const isValidSwipe = absX > swipeThreshold || absY > swipeThreshold;
      
      if (isValidSwipe) {
        // 水平滑动优先
        if (absX > absY) {
          if (deltaX > 0) {
            onSwipeRight?.();
          } else {
            onSwipeLeft?.();
          }
        } else {
          // 垂直滑动
          if (deltaY > 0) {
            onSwipeDown?.();
          } else {
            onSwipeUp?.();
          }
        }
      } else {
        // 记录点击位置用于双击检测
        if (onDoubleTap && !isLongPressing) {
          lastTapRef.current = touchStartRef.current;
        }
      }
    }
  }, [onSwipeLeft, onSwipeRight, onSwipeUp, onSwipeDown, onPinchEnd, onDoubleTap, swipeThreshold, isLongPressing]);

  return {
    onTouchStart,
    onTouchMove,
    onTouchEnd,
    isLongPressing,
  };
}

// 商品图片画廊手势使用示例
// components/products/ProductGallery.tsx
'use client';
import { useState, useRef } from 'react';
import Image from 'next/image';
import { useTouchGestures } from '@/hooks/use-touch-gestures';
import { cn } from '@/lib/utils';
import { ChevronLeft, ChevronRight, ZoomIn } from 'lucide-react';
import { Button } from '@/components/ui/button';

interface ProductGalleryProps {
  images: string[];
  productName: string;
  video?: string;
}

export function ProductGallery({ images, productName, video }: ProductGalleryProps) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [isZoomed, setIsZoomed] = useState(false);
  const [zoomScale, setZoomScale] = useState(1);
  const [zoomTransform, setZoomTransform] = useState({ x: 0, y: 0 });
  
  const containerRef = useRef<HTMLDivElement>(null);

  // 手势处理
  const gestureHandlers = useTouchGestures({
    onSwipeLeft: () => {
      if (!isZoomed && currentIndex < images.length - 1) {
        setCurrentIndex(prev => prev + 1);
      }
    },
    onSwipeRight: () => {
      if (!isZoomed && currentIndex > 0) {
        setCurrentIndex(prev => prev - 1);
      }
    },
    onDoubleTap: (point) => {
      if (isZoomed) {
        // 退出缩放
        setIsZoomed(false);
        setZoomScale(1);
        setZoomTransform({ x: 0, y: 0 });
      } else {
        // 进入缩放模式，以点击点为中心
        setIsZoomed(true);
        setZoomScale(2);
        
        if (containerRef.current) {
          const rect = containerRef.current.getBoundingClientRect();
          const centerX = rect.width / 2;
          const centerY = rect.height / 2;
          const offsetX = (centerX - point.x) * 2;
          const offsetY = (centerY - point.y) * 2;
          setZoomTransform({ x: offsetX, y: offsetY });
        }
      }
    },
    onPinch: (scale, velocity) => {
      if (scale > 1.1) {
        setIsZoomed(true);
        setZoomScale(Math.min(scale * 2, 4)); // 最大4倍缩放
      } else if (scale < 0.9) {
        if (zoomScale <= 1.2) {
          setIsZoomed(false);
          setZoomScale(1);
          setZoomTransform({ x: 0, y: 0 });
        } else {
          setZoomScale(Math.max(scale * 2, 1));
        }
      }
    },
    onPinchEnd: () => {
      if (zoomScale < 1.2) {
        setIsZoomed(false);
        setZoomScale(1);
        setZoomTransform({ x: 0, y: 0 });
      }
    },
    onLongPress: () => {
      // 长按显示操作菜单
      // 这里可以实现保存图片等功能
    },
    swipeThreshold: 50,
    doubleTapThreshold: 300,
    longPressThreshold: 800,
  });

  const goToPrevious = () => {
    setCurrentIndex(prev => Math.max(0, prev - 1));
  };

  const goToNext = () => {
    setCurrentIndex(prev => Math.min(images.length - 1, prev + 1));
  };

  return (
    <div className="relative w-full aspect-square bg-gray-100 rounded-lg overflow-hidden">
      {/* 主图片容器 */}
      <div
        ref={containerRef}
        className="relative w-full h-full overflow-hidden"
        {...gestureHandlers}
      >
        <div 
          className={cn(
            "w-full h-full transition-transform duration-300 ease-out",
            isZoomed && "cursor-move"
          )}
          style={{
            transform: `scale(${zoomScale}) translate(${zoomTransform.x}px, ${zoomTransform.y}px)`,
          }}
        >
          <Image
            src={images[currentIndex]}
            alt={`${productName} - 图片 ${currentIndex + 1}`}
            fill
            className="object-cover"
            sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
            priority={currentIndex === 0}
            quality={isZoomed ? 100 : 80}
          />
        </div>

        {/* 缩放指示器 */}
        {isZoomed && (
          <div className="absolute top-4 right-4 bg-black/50 text-white px-2 py-1 rounded text-sm">
            {Math.round(zoomScale * 100)}%
          </div>
        )}
      </div>

      {/* 导航按钮 */}
      {!isZoomed && images.length > 1 && (
        <>
          <Button
            variant="ghost"
            size="sm"
            className="absolute left-2 top-1/2 -translate-y-1/2 bg-white/80 hover:bg-white"
            onClick={goToPrevious}
            disabled={currentIndex === 0}
          >
            <ChevronLeft className="h-4 w-4" />
          </Button>
          
          <Button
            variant="ghost"
            size="sm"
            className="absolute right-2 top-1/2 -translate-y-1/2 bg-white/80 hover:bg-white"
            onClick={goToNext}
            disabled={currentIndex === images.length - 1}
          >
            <ChevronRight className="h-4 w-4" />
          </Button>
        </>
      )}

      {/* 图片指示器 */}
      {!isZoomed && images.length > 1 && (
        <div className="absolute bottom-4 left-1/2 -translate-x-1/2 flex space-x-2">
          {images.map((_, index) => (
            <button
              key={index}
              onClick={() => setCurrentIndex(index)}
              className={cn(
                "w-2 h-2 rounded-full transition-all duration-200",
                index === currentIndex 
                  ? "bg-white w-6" 
                  : "bg-white/50 hover:bg-white/80"
              )}
              aria-label={`查看第${index + 1}张图片`}
            />
          ))}
        </div>
      )}

      {/* 缩放按钮 */}
      {!isZoomed && (
        <Button
          variant="ghost"
          size="sm"
          className="absolute top-4 right-4 bg-white/80 hover:bg-white"
          onClick={() => {
            setIsZoomed(true);
            setZoomScale(2);
          }}
        >
          <ZoomIn className="h-4 w-4" />
        </Button>
      )}

      {/* 图片计数 */}
      <div className="absolute top-4 left-4 bg-black/50 text-white px-2 py-1 rounded text-sm">
        {currentIndex + 1} / {images.length}
      </div>
    </div>
  );
}
```

### 6.4 PWA安装和更新管理

#### PWA安装提示组件
```typescript
// components/pwa/InstallPrompt.tsx
'use client';
import { useState, useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { X, Download, Smartphone, Monitor } from 'lucide-react';
import { motion, AnimatePresence } from 'framer-motion';
import { useLocalStorage } from '@/hooks/use-local-storage';

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>;
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>;
}

export function InstallPrompt() {
  const [deferredPrompt, setDeferredPrompt] = useState<BeforeInstallPromptEvent | null>(null);
  const [showPrompt, setShowPrompt] = useState(false);
  const [isInstalled, setIsInstalled] = useState(false);
  const [deviceType, setDeviceType] = useState<'mobile' | 'desktop'>('mobile');
  const [dismissed, setDismissed] = useLocalStorage('pwa-install-dismissed', false);
  const [installCount, setInstallCount] = useLocalStorage('pwa-install-count', 0);

  useEffect(() => {
    // 检查设备类型
    const isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
      navigator.userAgent
    );
    setDeviceType(isMobile ? 'mobile' : 'desktop');

    // 检查是否已安装
    const checkIfInstalled = () => {
      const isStandalone = window.matchMedia('(display-mode: standalone)').matches;
      const isInApp = 'standalone' in window.navigator || (window.navigator as any).standalone;
      setIsInstalled(isStandalone || isInApp);
    };

    checkIfInstalled();

    // 监听安装提示事件
    const handleBeforeInstallPrompt = (e: Event) => {
      e.preventDefault();
      const event = e as BeforeInstallPromptEvent;
      setDeferredPrompt(event);
      
      // 智能显示策略
      const shouldShow = shouldShowInstallPrompt();
      if (shouldShow) {
        setTimeout(() => setShowPrompt(true), 2000); // 延迟2秒显示
      }
    };

    // 监听安装完成事件
    const handleAppInstalled = () => {
      setIsInstalled(true);
      setShowPrompt(false);
      setDeferredPrompt(null);
      
      // 显示安装成功提示
      if ('serviceWorker' in navigator) {
        navigator.serviceWorker.ready.then((registration) => {
          registration.showNotification('安装成功！', {
            body: 'ShopXO已成功安装到您的设备',
            icon: '/icons/icon-192x192.png',
            badge: '/icons/badge-72x72.png',
          });
        });
      }
    };

    window.addEventListener('beforeinstallprompt', handleBeforeInstallPrompt);
    window.addEventListener('appinstalled', handleAppInstalled);

    return () => {
      window.removeEventListener('beforeinstallprompt', handleBeforeInstallPrompt);
      window.removeEventListener('appinstalled', handleAppInstalled);
    };
  }, []);

  // 判断是否应该显示安装提示
  const shouldShowInstallPrompt = (): boolean => {
    if (isInstalled || dismissed) return false;
    
    // 根据访问次数决定
    const visitCount = parseInt(localStorage.getItem('visit-count') || '0');
    if (visitCount < 3) return false; // 至少访问3次后才提示
    
    // 根据用户行为决定
    const hasAddedToCart = localStorage.getItem('has-added-to-cart') === 'true';
    const hasViewedProduct = localStorage.getItem('has-viewed-product') === 'true';
    
    return hasAddedToCart || hasViewedProduct;
  };

  // 处理安装
  const handleInstall = async () => {
    if (!deferredPrompt) return;

    try {
      await deferredPrompt.prompt();
      const { outcome } = await deferredPrompt.userChoice;
      
      setInstallCount(installCount + 1);
      
      if (outcome === 'accepted') {
        console.log('用户安装了PWA');
        // 记录安装事件
        trackEvent('pwa_install_accepted', {
          device_type: deviceType,
          install_count: installCount + 1,
        });
      } else {
        console.log('用户取消了PWA安装');
        trackEvent('pwa_install_dismissed', {
          device_type: deviceType,
          install_count: installCount + 1,
        });
      }
      
      setDeferredPrompt(null);
      setShowPrompt(false);
    } catch (error) {
      console.error('安装失败:', error);
    }
  };

  // 处理忽略
  const handleDismiss = () => {
    setShowPrompt(false);
    setDismissed(true);
    
    trackEvent('pwa_install_later', {
      device_type: deviceType,
      install_count: installCount,
    });
  };

  // 永久忽略
  const handleNeverShow = () => {
    setShowPrompt(false);
    setDismissed(true);
    localStorage.setItem('pwa-install-never', 'true');
    
    trackEvent('pwa_install_never', {
      device_type: deviceType,
      install_count: installCount,
    });
  };

  // 分享安装链接(iOS Safari回退方案)
  const handleShareInstall = async () => {
    if (navigator.share) {
      try {
        await navigator.share({
          title: 'ShopXO 商城',
          text: '在手机上安装ShopXO，享受更好的购物体验！',
          url: window.location.href,
        });
      } catch (error) {
        console.log('用户取消了分享');
      }
    }
  };

  // 事件追踪
  const trackEvent = (eventName: string, properties: Record<string, any>) => {
    // 发送到分析服务
    if (window.gtag) {
      window.gtag('event', eventName, properties);
    }
  };

  // iOS Safari特殊处理
  const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
  const isSafari = /^((?!chrome|android).)*safari/i.test(navigator.userAgent);
  const isIOSSafari = isIOS && isSafari;

  if (isInstalled) return null;

  return (
    <AnimatePresence>
      {showPrompt && (
        <motion.div
          initial={{ y: 100, opacity: 0 }}
          animate={{ y: 0, opacity: 1 }}
          exit={{ y: 100, opacity: 0 }}
          transition={{ type: 'spring', damping: 25, stiffness: 200 }}
          className="fixed bottom-20 left-4 right-4 z-50 md:left-auto md:right-4 md:w-96"
        >
          <div className="bg-white rounded-xl shadow-2xl border p-6 relative overflow-hidden">
            {/* 背景装饰 */}
            <div className="absolute top-0 right-0 w-32 h-32 bg-gradient-to-bl from-primary-100 to-transparent rounded-full -mr-16 -mt-16" />
            
            {/* 关闭按钮 */}
            <Button
              variant="ghost"
              size="sm"
              onClick={handleDismiss}
              className="absolute top-2 right-2 p-1 h-auto z-10"
            >
              <X className="w-4 h-4" />
            </Button>

            <div className="flex items-start space-x-4">
              {/* 图标 */}
              <div className="flex-shrink-0 w-12 h-12 bg-primary-100 rounded-xl flex items-center justify-center">
                {deviceType === 'mobile' ? (
                  <Smartphone className="w-6 h-6 text-primary-600" />
                ) : (
                  <Monitor className="w-6 h-6 text-primary-600" />
                )}
              </div>

              <div className="flex-1 min-w-0">
                <h3 className="font-semibold text-gray-900 mb-1">
                  安装ShopXO到{deviceType === 'mobile' ? '手机' : '桌面'}
                </h3>
                <p className="text-sm text-gray-600 mb-4 leading-relaxed">
                  获得原生App般的体验：快速启动、离线浏览、推送通知
                </p>

                <div className="space-y-2">
                  {/* 安装按钮 */}
                  {deferredPrompt && !isIOSSafari ? (
                    <Button 
                      onClick={handleInstall}
                      className="w-full h-10"
                    >
                      <Download className="mr-2 h-4 w-4" />
                      立即安装
                    </Button>
                  ) : isIOSSafari ? (
                    <div className="text-sm text-gray-600 space-y-2">
                      <p>在Safari中：</p>
                      <ol className="list-decimal list-inside space-y-1 text-xs">
                        <li>点击底部 <span className="inline-flex items-center px-1">📤</span> 分享按钮</li>
                        <li>选择"添加到主屏幕"</li>
                        <li>点击"添加"完成安装</li>
                      </ol>
                    </div>
                  ) : (
                    <Button 
                      onClick={handleShareInstall}
                      variant="outline"
                      className="w-full h-10"
                    >
                      分享安装链接
                    </Button>
                  )}

                  {/* 次要操作 */}
                  <div className="flex space-x-2 text-xs">
                    <Button 
                      onClick={handleDismiss}
                      variant="ghost"
                      size="sm"
                      className="flex-1 h-8"
                    >
                      稍后再说
                    </Button>
                    <Button 
                      onClick={handleNeverShow}
                      variant="ghost"
                      size="sm"
                      className="flex-1 h-8 text-gray-500"
                    >
                      不再提醒
                    </Button>
                  </div>
                </div>
              </div>
            </div>

            {/* 特性展示 */}
            <div className="mt-4 pt-4 border-t border-gray-100">
              <div className="grid grid-cols-3 gap-3 text-center">
                <div className="text-xs">
                  <div className="w-8 h-8 bg-green-100 rounded-lg flex items-center justify-center mx-auto mb-1">
                    <span className="text-green-600">🚀</span>
                  </div>
                  <span className="text-gray-600">快速启动</span>
                </div>
                <div className="text-xs">
                  <div className="w-8 h-8 bg-blue-100 rounded-lg flex items-center justify-center mx-auto mb-1">
                    <span className="text-blue-600">📱</span>
                  </div>
                  <span className="text-gray-600">离线浏览</span>
                </div>
                <div className="text-xs">
                  <div className="w-8 h-8 bg-purple-100 rounded-lg flex items-center justify-center mx-auto mb-1">
                    <span className="text-purple-600">🔔</span>
                  </div>
                  <span className="text-gray-600">消息推送</span>
                </div>
              </div>
            </div>
          </div>
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

#### PWA更新管理
```typescript
// hooks/use-pwa-update.ts
import { useState, useEffect } from 'react';
import { useToast } from '@/components/ui/use-toast';

export function usePWAUpdate() {
  const [hasUpdate, setHasUpdate] = useState(false);
  const [isUpdating, setIsUpdating] = useState(false);
  const [registration, setRegistration] = useState<ServiceWorkerRegistration | null>(null);
  const { toast } = useToast();

  useEffect(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.ready.then((reg) => {
        setRegistration(reg);
        
        // 检查更新
        reg.addEventListener('updatefound', () => {
          const newWorker = reg.installing;
          if (newWorker) {
            newWorker.addEventListener('statechange', () => {
              if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                setHasUpdate(true);
                
                toast({
                  title: '发现新版本',
                  description: '点击更新获取最新功能',
                  action: (
                    <Button onClick={updateApp} variant="outline" size="sm">
                      立即更新
                    </Button>
                  ),
                  duration: 10000,
                });
              }
            });
          }
        });

        // 定期检查更新
        const checkForUpdates = () => {
          reg.update();
        };

        // 每10分钟检查一次更新
        const updateInterval = setInterval(checkForUpdates, 10 * 60 * 1000);
        
        // 页面聚焦时检查更新
        const handleFocus = () => {
          checkForUpdates();
        };
        
        window.addEventListener('focus', handleFocus);
        
        return () => {
          clearInterval(updateInterval);
          window.removeEventListener('focus', handleFocus);
        };
      });
    }
  }, []);

  const updateApp = async () => {
    if (!registration || !hasUpdate) return;

    setIsUpdating(true);
    
    try {
      // 告诉等待中的SW跳过等待并激活
      if (registration.waiting) {
        registration.waiting.postMessage({ type: 'SKIP_WAITING' });
        
        // 监听控制权变更
        const handleControllerChange = () => {
          window.location.reload();
        };
        
        navigator.serviceWorker.addEventListener('controllerchange', handleControllerChange);
        
        // 5秒后如果还没更新就强制刷新
        setTimeout(() => {
          navigator.serviceWorker.removeEventListener('controllerchange', handleControllerChange);
          window.location.reload();
        }, 5000);
      }
    } catch (error) {
      console.error('更新失败:', error);
      toast({
        title: '更新失败',
        description: '请稍后重试或刷新页面',
        variant: 'destructive',
      });
    } finally {
      setIsUpdating(false);
    }
  };

  return {
    hasUpdate,
    isUpdating,
    updateApp,
  };
}

// components/pwa/UpdateNotification.tsx
'use client';
import { motion, AnimatePresence } from 'framer-motion';
import { Button } from '@/components/ui/button';
import { RefreshCw, X } from 'lucide-react';
import { usePWAUpdate } from '@/hooks/use-pwa-update';
import { useState } from 'react';

export function UpdateNotification() {
  const { hasUpdate, isUpdating, updateApp } = usePWAUpdate();
  const [isDismissed, setIsDismissed] = useState(false);

  if (!hasUpdate || isDismissed) return null;

  return (
    <AnimatePresence>
      <motion.div
        initial={{ y: -100, opacity: 0 }}
        animate={{ y: 0, opacity: 1 }}
        exit={{ y: -100, opacity: 0 }}
        className="fixed top-4 left-4 right-4 z-50 md:left-auto md:right-4 md:w-80"
      >
        <div className="bg-primary-600 text-white rounded-lg shadow-lg p-4">
          <div className="flex items-center justify-between">
            <div className="flex items-center space-x-3">
              <RefreshCw className={cn(
                "w-5 h-5",
                isUpdating && "animate-spin"
              )} />
              <div>
                <h4 className="font-semibold">新版本可用</h4>
                <p className="text-sm text-primary-100">
                  点击更新获取最新功能和性能改进
                </p>
              </div>
            </div>
            
            <Button
              variant="ghost"
              size="sm"
              onClick={() => setIsDismissed(true)}
              className="text-white hover:bg-primary-700 p-1 h-auto"
            >
              <X className="w-4 h-4" />
            </Button>
          </div>
          
          <div className="mt-3 flex space-x-2">
            <Button
              onClick={updateApp}
              disabled={isUpdating}
              variant="secondary"
              size="sm"
              className="flex-1"
            >
              {isUpdating ? '更新中...' : '立即更新'}
            </Button>
            
            <Button
              onClick={() => setIsDismissed(true)}
              variant="ghost"
              size="sm"
              className="text-white hover:bg-primary-700"
            >
              稍后
            </Button>
          </div>
        </div>
      </motion.div>
    </AnimatePresence>
  );
}
```

这个PWA和移动端优化方案提供了：

1. **完整的PWA配置**：清单文件、Service Worker、离线支持
2. **智能安装提示**：基于用户行为的安装引导
3. **响应式设计系统**：移动优先的Tailwind配置
4. **原生级交互体验**：手势支持、触摸优化、动画效果
5. **自动更新机制**：后台更新检测和用户友好的更新提示
6. **离线功能**：缓存策略、后台同步、离线页面
7. **性能优化**：懒加载、代码分割、图片优化

整个方案确保了在移动设备上获得接近原生App的用户体验。

---

文档已完成，现在为您创建最后的总结文档...