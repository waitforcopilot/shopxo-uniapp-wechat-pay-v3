# ShopXO重构实施详解 - 技术实现细节

## 第四部分：服务端组件与客户端组件策略

### 4.1 组件渲染策略决策

```typescript
// lib/component-strategy.ts
export enum ComponentRenderType {
  SERVER_ONLY = 'server',      // 纯服务端渲染
  CLIENT_ONLY = 'client',      // 纯客户端渲染
  HYBRID = 'hybrid',           // 混合渲染
  ISLAND = 'island'            // 岛屿架构
}

export interface ComponentConfig {
  renderType: ComponentRenderType;
  cacheable: boolean;
  revalidate?: number;
  dependencies?: string[];
  preload?: boolean;
  lazy?: boolean;
}

// 组件配置映射
export const COMPONENT_CONFIGS: Record<string, ComponentConfig> = {
  // 服务端组件 - 静态内容、SEO重要
  ProductInfo: {
    renderType: ComponentRenderType.SERVER_ONLY,
    cacheable: true,
    revalidate: 3600, // 1小时
  },
  
  CategoryList: {
    renderType: ComponentRenderType.SERVER_ONLY,
    cacheable: true,
    revalidate: 1800, // 30分钟
  },
  
  ProductDescription: {
    renderType: ComponentRenderType.SERVER_ONLY,
    cacheable: true,
    revalidate: 86400, // 24小时
  },
  
  // 客户端组件 - 交互功能
  AddToCartButton: {
    renderType: ComponentRenderType.CLIENT_ONLY,
    cacheable: false,
    dependencies: ['cart-store'],
  },
  
  SearchBox: {
    renderType: ComponentRenderType.CLIENT_ONLY,
    cacheable: false,
    preload: true,
  },
  
  UserDropdown: {
    renderType: ComponentRenderType.CLIENT_ONLY,
    cacheable: false,
    dependencies: ['auth-store'],
  },
  
  // 混合组件 - 首屏SSR + 客户端交互
  ProductList: {
    renderType: ComponentRenderType.HYBRID,
    cacheable: true,
    revalidate: 600, // 10分钟
    lazy: true,
  },
  
  ReviewList: {
    renderType: ComponentRenderType.HYBRID,
    cacheable: true,
    revalidate: 1800,
    lazy: true,
  },
};
```

### 4.2 商品详情页实现

#### 服务端组件实现
```typescript
// app/[locale]/products/[id]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { Metadata } from 'next';
import { getProduct, getRelatedProducts, getProductReviews } from '@/lib/api/products';
import { ProductGallery } from './components/ProductGallery';
import { ProductInfo } from './components/ProductInfo';
import { ProductTabs } from './components/ProductTabs';
import { RelatedProducts } from './components/RelatedProducts';
import { AddToCartSection } from './components/AddToCartSection';
import { ProductSkeleton } from './components/ProductSkeleton';
import { BreadcrumbNav } from '@/components/navigation/BreadcrumbNav';

interface ProductPageProps {
  params: { 
    id: string; 
    locale: string;
  };
  searchParams: { 
    variant?: string;
    from?: string;
  };
}

// 服务端组件 - 商品详情页主页面
export default async function ProductPage({ params, searchParams }: ProductPageProps) {
  // 并行获取数据
  const [product, relatedProducts] = await Promise.all([
    getProduct(params.id, {
      includeReviews: true,
      includeVariants: true,
      includeBrand: true,
      includeCategory: true,
    }),
    getRelatedProducts(params.id, { limit: 8 })
  ]);

  if (!product) {
    notFound();
  }

  // 结构化数据用于SEO
  const structuredData = {
    "@context": "https://schema.org",
    "@type": "Product",
    "name": product.title,
    "image": product.images,
    "description": product.description,
    "brand": {
      "@type": "Brand",
      "name": product.brand?.name
    },
    "offers": {
      "@type": "Offer",
      "price": product.price,
      "priceCurrency": "CNY",
      "availability": product.stock > 0 ? "https://schema.org/InStock" : "https://schema.org/OutOfStock"
    },
    "aggregateRating": product.reviews?.length > 0 ? {
      "@type": "AggregateRating",
      "ratingValue": product.averageRating,
      "reviewCount": product.reviews.length
    } : undefined
  };

  return (
    <>
      {/* 结构化数据 */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(structuredData) }}
      />

      <div className="container mx-auto px-4 py-6">
        {/* 面包屑导航 - 服务端组件 */}
        <BreadcrumbNav 
          items={[
            { label: '首页', href: '/' },
            { label: product.category.name, href: `/category/${product.category.slug}` },
            { label: product.title }
          ]} 
        />

        {/* 商品主要信息区域 */}
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-8 mt-6">
          {/* 商品图片画廊 - 客户端组件(需要交互) */}
          <ProductGallery 
            images={product.images}
            video={product.video}
            productName={product.title}
          />
          
          {/* 商品信息和购买区域 */}
          <div className="space-y-6">
            {/* 基本信息 - 服务端组件 */}
            <ProductInfo product={product} />
            
            {/* 规格选择和购买 - 客户端组件 */}
            <Suspense fallback={<div className="h-32 bg-gray-100 animate-pulse rounded" />}>
              <AddToCartSection 
                product={product}
                selectedVariant={searchParams.variant}
              />
            </Suspense>
          </div>
        </div>

        {/* 商品详情标签页 - 混合组件 */}
        <Suspense fallback={<ProductSkeleton />}>
          <ProductTabs 
            description={product.description}
            specifications={product.specifications}
            reviewsCount={product.reviews?.length || 0}
            productId={product.id}
          />
        </Suspense>

        {/* 相关商品 - 服务端组件 */}
        {relatedProducts.length > 0 && (
          <section className="mt-12">
            <h2 className="text-2xl font-bold mb-6">相关商品推荐</h2>
            <RelatedProducts products={relatedProducts} />
          </section>
        )}
      </div>
    </>
  );
}

// 元数据生成 - 服务端
export async function generateMetadata({ params }: ProductPageProps): Promise<Metadata> {
  const product = await getProduct(params.id, { includeBrand: true, includeCategory: true });
  
  if (!product) {
    return {
      title: '商品未找到 - ShopXO',
      description: '抱歉，您查找的商品不存在。',
    };
  }

  const title = product.seoTitle || `${product.title} - ${product.brand?.name || 'ShopXO'}`;
  const description = product.seoDescription || product.subtitle || product.description?.slice(0, 160);
  
  return {
    title,
    description,
    keywords: product.seoKeywords,
    openGraph: {
      title,
      description,
      images: [
        {
          url: product.images[0],
          width: 800,
          height: 800,
          alt: product.title,
        }
      ],
      type: 'website',
      siteName: 'ShopXO',
    },
    twitter: {
      card: 'summary_large_image',
      title,
      description,
      images: [product.images[0]],
    },
    other: {
      'product:price:amount': product.price.toString(),
      'product:price:currency': 'CNY',
      'product:availability': product.stock > 0 ? 'in stock' : 'out of stock',
      'product:condition': 'new',
      'product:retailer': 'ShopXO',
    },
  };
}

// 静态参数生成(可选，用于预生成热门商品页面)
export async function generateStaticParams() {
  // 获取热门商品ID列表
  const popularProducts = await getPopularProducts({ limit: 100 });
  
  return popularProducts.map((product) => ({
    id: product.id,
  }));
}
```

#### 客户端交互组件
```typescript
// components/products/AddToCartSection.tsx
'use client';
import { useState, useEffect, useCallback } from 'react';
import { useCartStore } from '@/store/cart';
import { useAuth } from '@/hooks/use-auth';
import { useToast } from '@/components/ui/use-toast';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { ShoppingCart, Plus, Minus, Heart, Share2 } from 'lucide-react';
import { QuantitySelector } from './QuantitySelector';
import { VariantSelector } from './VariantSelector';
import { PriceDisplay } from './PriceDisplay';
import { StockIndicator } from './StockIndicator';
import { motion, AnimatePresence } from 'framer-motion';

interface AddToCartSectionProps {
  product: Product;
  selectedVariant?: string;
}

export function AddToCartSection({ product, selectedVariant }: AddToCartSectionProps) {
  const [selectedSku, setSelectedSku] = useState<ProductSku | null>(null);
  const [quantity, setQuantity] = useState(1);
  const [isAddingToCart, setIsAddingToCart] = useState(false);
  const [isFavorited, setIsFavorited] = useState(false);
  
  const { isAuthenticated } = useAuth();
  const { addItem, items } = useCartStore();
  const { toast } = useToast();

  // 初始化选中的SKU
  useEffect(() => {
    if (selectedVariant && product.skus) {
      const sku = product.skus.find(s => s.id === selectedVariant);
      setSelectedSku(sku || product.skus[0] || null);
    } else if (product.skus?.length > 0) {
      setSelectedSku(product.skus[0]);
    }
  }, [selectedVariant, product.skus]);

  // 检查当前商品是否已在购物车
  const cartItem = items.find(item => 
    item.productId === product.id && 
    item.skuId === selectedSku?.id
  );
  const cartQuantity = cartItem?.quantity || 0;

  // 计算可购买的最大数量
  const maxQuantity = Math.min(
    (selectedSku?.stock || product.stock) - cartQuantity,
    10 // 限制单次最大购买数量
  );

  // 处理SKU选择
  const handleSkuChange = useCallback((sku: ProductSku) => {
    setSelectedSku(sku);
    setQuantity(1); // 重置数量
    
    // 更新URL参数
    const url = new URL(window.location.href);
    url.searchParams.set('variant', sku.id);
    window.history.replaceState({}, '', url.toString());
  }, []);

  // 添加到购物车
  const handleAddToCart = useCallback(async () => {
    if (!selectedSku) {
      toast({
        title: '请选择商品规格',
        variant: 'destructive',
      });
      return;
    }

    if (quantity > maxQuantity) {
      toast({
        title: '购买数量超限',
        description: `最多可购买 ${maxQuantity} 件`,
        variant: 'destructive',
      });
      return;
    }

    setIsAddingToCart(true);

    try {
      // 添加到本地状态
      addItem({
        productId: product.id,
        skuId: selectedSku.id,
        quantity,
        price: selectedSku.price,
        productTitle: product.title,
        productImage: product.images[0],
        skuSpecs: selectedSku.specifications,
      });

      // 同步到服务器(如果用户已登录)
      if (isAuthenticated) {
        await fetch('/api/cart/add', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            productId: product.id,
            skuId: selectedSku.id,
            quantity,
          }),
        });
      }

      toast({
        title: '已添加到购物车',
        description: `${product.title} × ${quantity}`,
      });

      // 触发添加到购物车的动画效果
      // 可以通过事件或状态管理器通知其他组件
      
    } catch (error) {
      toast({
        title: '添加失败',
        description: '网络错误，请稍后重试',
        variant: 'destructive',
      });
    } finally {
      setIsAddingToCart(false);
    }
  }, [product, selectedSku, quantity, maxQuantity, isAuthenticated, addItem, toast]);

  // 立即购买
  const handleBuyNow = useCallback(async () => {
    await handleAddToCart();
    // 跳转到结算页面
    window.location.href = '/checkout';
  }, [handleAddToCart]);

  // 切换收藏状态
  const handleToggleFavorite = useCallback(async () => {
    if (!isAuthenticated) {
      toast({
        title: '请先登录',
        description: '登录后可以收藏商品',
        variant: 'destructive',
      });
      return;
    }

    try {
      const response = await fetch('/api/favorites/toggle', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productId: product.id }),
      });

      if (response.ok) {
        setIsFavorited(!isFavorited);
        toast({
          title: isFavorited ? '已取消收藏' : '已添加收藏',
        });
      }
    } catch (error) {
      toast({
        title: '操作失败',
        description: '请稍后重试',
        variant: 'destructive',
      });
    }
  }, [isAuthenticated, isFavorited, product.id, toast]);

  // 分享商品
  const handleShare = useCallback(async () => {
    if (navigator.share) {
      try {
        await navigator.share({
          title: product.title,
          text: product.subtitle,
          url: window.location.href,
        });
      } catch (error) {
        // 用户取消分享
      }
    } else {
      // 回退到复制链接
      navigator.clipboard.writeText(window.location.href);
      toast({
        title: '链接已复制',
        description: '您可以粘贴分享给朋友',
      });
    }
  }, [product, toast]);

  const currentPrice = selectedSku?.price || product.price;
  const originalPrice = selectedSku?.originalPrice || product.originalPrice;
  const currentStock = selectedSku?.stock || product.stock;

  return (
    <div className="space-y-6 p-6 bg-white rounded-lg shadow-sm border">
      {/* 价格显示 */}
      <PriceDisplay 
        price={currentPrice}
        originalPrice={originalPrice}
        currency="¥"
      />

      {/* 库存状态 */}
      <StockIndicator 
        stock={currentStock}
        lowStockThreshold={10}
      />

      {/* SKU选择器 */}
      {product.skus && product.skus.length > 0 && (
        <VariantSelector
          skus={product.skus}
          selectedSku={selectedSku}
          onSkuChange={handleSkuChange}
        />
      )}

      {/* 数量选择 */}
      <div className="flex items-center space-x-4">
        <span className="text-sm font-medium text-gray-700">数量:</span>
        <QuantitySelector
          quantity={quantity}
          maxQuantity={maxQuantity}
          onChange={setQuantity}
          disabled={currentStock === 0}
        />
        {cartQuantity > 0 && (
          <span className="text-xs text-gray-500">
            购物车中已有 {cartQuantity} 件
          </span>
        )}
      </div>

      {/* 主要操作按钮 */}
      <div className="space-y-3">
        <motion.div
          whileTap={{ scale: 0.98 }}
          className="flex space-x-3"
        >
          <Button
            onClick={handleAddToCart}
            disabled={isAddingToCart || currentStock === 0 || quantity > maxQuantity}
            className="flex-1 h-12"
            size="lg"
          >
            <ShoppingCart className="mr-2 h-5 w-5" />
            {isAddingToCart ? '添加中...' : currentStock === 0 ? '缺货' : '加入购物车'}
          </Button>
          
          <Button
            onClick={handleBuyNow}
            disabled={isAddingToCart || currentStock === 0}
            variant="outline"
            className="flex-1 h-12"
            size="lg"
          >
            立即购买
          </Button>
        </motion.div>

        {/* 次要操作按钮 */}
        <div className="flex space-x-3">
          <Button
            onClick={handleToggleFavorite}
            variant="ghost"
            size="sm"
            className="flex-1"
          >
            <Heart className={`mr-2 h-4 w-4 ${isFavorited ? 'fill-red-500 text-red-500' : ''}`} />
            {isFavorited ? '已收藏' : '收藏'}
          </Button>
          
          <Button
            onClick={handleShare}
            variant="ghost"
            size="sm"
            className="flex-1"
          >
            <Share2 className="mr-2 h-4 w-4" />
            分享
          </Button>
        </div>
      </div>

      {/* 服务保障 */}
      <div className="pt-4 border-t border-gray-200">
        <div className="grid grid-cols-2 gap-3 text-xs text-gray-600">
          <div className="flex items-center space-x-2">
            <Badge variant="secondary" className="text-xs">✓</Badge>
            <span>7天无理由退货</span>
          </div>
          <div className="flex items-center space-x-2">
            <Badge variant="secondary" className="text-xs">✓</Badge>
            <span>48小时发货</span>
          </div>
          <div className="flex items-center space-x-2">
            <Badge variant="secondary" className="text-xs">✓</Badge>
            <span>正品保证</span>
          </div>
          <div className="flex items-center space-x-2">
            <Badge variant="secondary" className="text-xs">✓</Badge>
            <span>全国包邮</span>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 4.3 混合渲染策略实现

#### 商品列表混合组件
```typescript
// components/products/ProductList.tsx (服务端入口)
import { Suspense } from 'react';
import { getProducts } from '@/lib/api/products';
import { ProductListClient } from './ProductListClient';
import { ProductListSkeleton } from './ProductListSkeleton';
import { ProductCard } from './ProductCard';

interface ProductListProps {
  categoryId?: string;
  searchTerm?: string;
  sortBy?: string;
  priceRange?: [number, number];
  brands?: string[];
  page?: number;
  limit?: number;
  enableInfiniteScroll?: boolean;
}

// 服务端组件：首次渲染
export async function ProductList({ 
  categoryId, 
  searchTerm, 
  sortBy = 'default',
  priceRange,
  brands,
  page = 1, 
  limit = 20,
  enableInfiniteScroll = true 
}: ProductListProps) {
  // 服务端获取初始数据
  const initialData = await getProducts({
    categoryId,
    searchTerm,
    sortBy,
    priceRange,
    brands,
    page,
    limit,
    includeFilters: true, // 包含筛选器数据
  });

  // 如果没有启用无限滚动，直接渲染静态列表
  if (!enableInfiniteScroll) {
    return (
      <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {initialData.products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    );
  }

  // 启用无限滚动时，传递给客户端组件
  return (
    <Suspense fallback={<ProductListSkeleton count={limit} />}>
      <ProductListClient 
        initialData={initialData}
        searchParams={{
          categoryId,
          searchTerm,
          sortBy,
          priceRange,
          brands,
          limit,
        }}
      />
    </Suspense>
  );
}

// components/products/ProductListClient.tsx (客户端组件)
'use client';
import { useState, useCallback, useEffect } from 'react';
import { useInView } from 'react-intersection-observer';
import { useDebounce } from '@/hooks/use-debounce';
import { ProductCard } from './ProductCard';
import { ProductListSkeleton } from './ProductListSkeleton';
import { ProductFilters } from './ProductFilters';
import { ProductSorter } from './ProductSorter';
import { Button } from '@/components/ui/button';
import { RefreshCw } from 'lucide-react';

interface ProductListClientProps {
  initialData: ProductListResponse;
  searchParams: {
    categoryId?: string;
    searchTerm?: string;
    sortBy?: string;
    priceRange?: [number, number];
    brands?: string[];
    limit?: number;
  };
}

export function ProductListClient({ initialData, searchParams }: ProductListClientProps) {
  const [products, setProducts] = useState(initialData.products);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(initialData.pagination.hasNext);
  const [currentPage, setCurrentPage] = useState(1);
  const [filters, setFilters] = useState({
    sortBy: searchParams.sortBy || 'default',
    priceRange: searchParams.priceRange,
    brands: searchParams.brands || [],
  });
  const [error, setError] = useState<string | null>(null);

  // 防抖处理筛选条件变化
  const debouncedFilters = useDebounce(filters, 500);

  // 无限滚动监听器
  const { ref: loadMoreRef, inView } = useInView({
    threshold: 0,
    rootMargin: '100px',
  });

  // 加载更多商品
  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;
    
    setLoading(true);
    setError(null);
    
    try {
      const nextPage = currentPage + 1;
      const response = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...searchParams,
          ...debouncedFilters,
          page: nextPage,
        }),
      });

      if (!response.ok) {
        throw new Error('加载失败');
      }

      const data = await response.json();
      
      setProducts(prev => [...prev, ...data.products]);
      setHasMore(data.pagination.hasNext);
      setCurrentPage(nextPage);
      
    } catch (error) {
      setError('加载商品失败，请稍后重试');
      console.error('Load more products failed:', error);
    } finally {
      setLoading(false);
    }
  }, [searchParams, debouncedFilters, currentPage, loading, hasMore]);

  // 重新加载商品列表
  const reloadProducts = useCallback(async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...searchParams,
          ...debouncedFilters,
          page: 1,
        }),
      });

      if (!response.ok) {
        throw new Error('加载失败');
      }

      const data = await response.json();
      
      setProducts(data.products);
      setHasMore(data.pagination.hasNext);
      setCurrentPage(1);
      
    } catch (error) {
      setError('加载商品失败，请稍后重试');
      console.error('Reload products failed:', error);
    } finally {
      setLoading(false);
    }
  }, [searchParams, debouncedFilters]);

  // 监听筛选条件变化
  useEffect(() => {
    if (JSON.stringify(debouncedFilters) !== JSON.stringify({
      sortBy: searchParams.sortBy || 'default',
      priceRange: searchParams.priceRange,
      brands: searchParams.brands || [],
    })) {
      reloadProducts();
    }
  }, [debouncedFilters, reloadProducts]);

  // 监听无限滚动
  useEffect(() => {
    if (inView && hasMore && !loading) {
      loadMore();
    }
  }, [inView, hasMore, loading, loadMore]);

  return (
    <div className="space-y-6">
      {/* 筛选和排序 */}
      <div className="flex flex-col lg:flex-row lg:items-center lg:justify-between gap-4">
        <ProductFilters
          availableFilters={initialData.filters}
          currentFilters={filters}
          onFiltersChange={setFilters}
        />
        
        <ProductSorter
          currentSort={filters.sortBy}
          onSortChange={(sortBy) => setFilters(prev => ({ ...prev, sortBy }))}
        />
      </div>

      {/* 商品列表 */}
      <div>
        {products.length === 0 && !loading ? (
          <div className="text-center py-12">
            <p className="text-gray-500 mb-4">暂无匹配的商品</p>
            <Button onClick={reloadProducts} variant="outline">
              <RefreshCw className="mr-2 h-4 w-4" />
              重新加载
            </Button>
          </div>
        ) : (
          <>
            <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
              {products.map((product, index) => (
                <ProductCard 
                  key={`${product.id}-${index}`} 
                  product={product}
                  priority={index < 8} // 前8个商品优先加载
                />
              ))}
            </div>

            {/* 加载更多区域 */}
            {hasMore && (
              <div ref={loadMoreRef} className="py-8 text-center">
                {loading ? (
                  <ProductListSkeleton count={4} />
                ) : (
                  <Button onClick={loadMore} variant="outline">
                    点击加载更多
                  </Button>
                )}
              </div>
            )}

            {/* 没有更多商品提示 */}
            {!hasMore && products.length > 0 && (
              <div className="text-center py-8 text-gray-500 text-sm">
                — 已显示全部商品 —
              </div>
            )}
          </>
        )}
      </div>

      {/* 错误提示 */}
      {error && (
        <div className="bg-red-50 border border-red-200 rounded-lg p-4 text-center">
          <p className="text-red-600 text-sm mb-2">{error}</p>
          <Button onClick={reloadProducts} variant="outline" size="sm">
            重试
          </Button>
        </div>
      )}
    </div>
  );
}
```

---

## 第五部分：缓存策略和性能优化方案

### 5.1 多层缓存架构设计

```typescript
// lib/cache/cache-manager.ts
import { Redis } from 'ioredis';
import { unstable_cache as nextCache } from 'next/cache';
import { revalidateTag } from 'next/cache';

// Redis连接配置
export const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3,
  lazyConnect: true,
  connectTimeout: 10000,
  commandTimeout: 5000,
});

// 缓存层级定义
export enum CacheLayer {
  BROWSER = 'browser',     // 浏览器缓存
  CDN = 'cdn',            // CDN缓存
  NEXT = 'next',          // Next.js缓存
  REDIS = 'redis',        // Redis缓存
  DATABASE = 'database'   // 数据库
}

// 缓存策略配置
export interface CacheConfig {
  layers: CacheLayer[];
  ttl: {
    browser?: number;
    cdn?: number;
    next?: number;
    redis?: number;
  };
  tags?: string[];
  revalidate?: number;
  staleWhileRevalidate?: boolean;
}

// 缓存键生成器
export class CacheKeyGenerator {
  static product(id: string): string {
    return `product:${id}`;
  }

  static productList(params: Record<string, any>): string {
    const sortedParams = Object.keys(params)
      .sort()
      .map(key => `${key}:${params[key]}`)
      .join(',');
    return `products:${btoa(sortedParams)}`;
  }

  static category(id: string): string {
    return `category:${id}`;
  }

  static categoryTree(): string {
    return 'categories:tree';
  }

  static userCart(userId: string): string {
    return `cart:${userId}`;
  }

  static userProfile(userId: string): string {
    return `user:${userId}`;
  }

  static searchResults(query: string, filters: Record<string, any>): string {
    const filterStr = Object.keys(filters)
      .sort()
      .map(key => `${key}:${filters[key]}`)
      .join(',');
    return `search:${btoa(query)}:${btoa(filterStr)}`;
  }

  static diyPage(type: string, id?: string): string {
    return id ? `diy:${type}:${id}` : `diy:${type}:default`;
  }

  static pluginConfig(name: string): string {
    return `plugin:${name}:config`;
  }
}

// 缓存时间常量
export const CacheTTL = {
  VERY_SHORT: 5 * 60,        // 5分钟
  SHORT: 15 * 60,            // 15分钟
  MEDIUM: 60 * 60,           // 1小时
  LONG: 24 * 60 * 60,        // 24小时
  VERY_LONG: 7 * 24 * 60 * 60, // 7天
} as const;

// 统一缓存管理器
export class CacheManager {
  // Redis操作
  static async getFromRedis<T>(key: string): Promise<T | null> {
    try {
      const cached = await redis.get(key);
      return cached ? JSON.parse(cached) : null;
    } catch (error) {
      console.error('Redis get error:', error);
      return null;
    }
  }

  static async setToRedis<T>(
    key: string, 
    value: T, 
    ttl: number = CacheTTL.MEDIUM
  ): Promise<void> {
    try {
      await redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Redis set error:', error);
    }
  }

  static async deleteFromRedis(pattern: string): Promise<void> {
    try {
      const keys = await redis.keys(pattern);
      if (keys.length > 0) {
        await redis.del(...keys);
      }
    } catch (error) {
      console.error('Redis delete error:', error);
    }
  }

  // Next.js缓存操作
  static createNextjsCache<T extends any[], R>(
    fn: (...args: T) => Promise<R>,
    keyParts: string[],
    config: { revalidate?: number; tags?: string[] } = {}
  ) {
    return nextCache(fn, keyParts, {
      revalidate: config.revalidate,
      tags: config.tags,
    });
  }

  // 缓存穿透保护
  static async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    config: CacheConfig
  ): Promise<T> {
    // 尝试从Redis获取
    if (config.layers.includes(CacheLayer.REDIS)) {
      const cached = await this.getFromRedis<T>(key);
      if (cached !== null) {
        return cached;
      }
    }

    // 缓存未命中，执行fetcher
    try {
      const data = await fetcher();
      
      // 存储到Redis
      if (config.layers.includes(CacheLayer.REDIS) && config.ttl.redis) {
        await this.setToRedis(key, data, config.ttl.redis);
      }

      return data;
    } catch (error) {
      console.error('Cache fetcher error:', error);
      throw error;
    }
  }

  // 批量缓存失效
  static async invalidate(tags: string[]): Promise<void> {
    try {
      // Next.js缓存失效
      for (const tag of tags) {
        revalidateTag(tag);
      }

      // Redis缓存失效
      const patterns = tags.map(tag => `*${tag}*`);
      for (const pattern of patterns) {
        await this.deleteFromRedis(pattern);
      }
    } catch (error) {
      console.error('Cache invalidation error:', error);
    }
  }

  // 预热缓存
  static async warmup(): Promise<void> {
    const warmupTasks = [
      this.warmupCategories(),
      this.warmupHotProducts(),
      this.warmupDiyPages(),
    ];

    await Promise.allSettled(warmupTasks);
  }

  private static async warmupCategories(): Promise<void> {
    try {
      const { getCategories } = await import('@/lib/api/categories');
      await getCategories();
    } catch (error) {
      console.error('Categories warmup failed:', error);
    }
  }

  private static async warmupHotProducts(): Promise<void> {
    try {
      const { getHotProducts } = await import('@/lib/api/products');
      await getHotProducts({ limit: 20 });
    } catch (error) {
      console.error('Hot products warmup failed:', error);
    }
  }

  private static async warmupDiyPages(): Promise<void> {
    try {
      const { getDiyPage } = await import('@/lib/api/diy');
      await getDiyPage('home');
    } catch (error) {
      console.error('DIY pages warmup failed:', error);
    }
  }
}

// 缓存装饰器
export function Cached(config: CacheConfig) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const key = `${target.constructor.name}:${propertyKey}:${JSON.stringify(args)}`;
      
      return CacheManager.getOrSet(
        key,
        () => originalMethod.apply(this, args),
        config
      );
    };

    return descriptor;
  };
}
```

### 5.2 API数据获取优化

```typescript
// lib/api/products.ts
import { prisma } from '@/lib/db';
import { CacheManager, CacheKeyGenerator, CacheTTL, CacheLayer } from '@/lib/cache/cache-manager';

interface ProductQueryParams {
  categoryId?: string;
  searchTerm?: string;
  sortBy?: 'default' | 'price_asc' | 'price_desc' | 'sales' | 'rating' | 'newest';
  priceRange?: [number, number];
  brands?: string[];
  page?: number;
  limit?: number;
  includeReviews?: boolean;
  includeVariants?: boolean;
  includeBrand?: boolean;
  includeCategory?: boolean;
  includeFilters?: boolean;
}

interface ProductListResponse {
  products: Product[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
  filters?: {
    brands: Array<{ id: string; name: string; count: number }>;
    priceRanges: Array<{ min: number; max: number; count: number }>;
    categories: Array<{ id: string; name: string; count: number }>;
  };
}

// 获取单个商品 - 高缓存
export const getProduct = CacheManager.createNextjsCache(
  async (id: string, options: Partial<ProductQueryParams> = {}): Promise<Product | null> => {
    const cacheKey = CacheKeyGenerator.product(id);
    
    return CacheManager.getOrSet(
      cacheKey,
      async () => {
        const product = await prisma.product.findUnique({
          where: { 
            id,
            status: 'ACTIVE' 
          },
          include: {
            category: options.includeCategory,
            brand: options.includeBrand,
            skus: options.includeVariants ? {
              where: { status: 'ACTIVE' },
              orderBy: { price: 'asc' }
            } : false,
            reviews: options.includeReviews ? {
              where: { status: 'APPROVED' },
              take: 10,
              orderBy: { createdAt: 'desc' },
              include: {
                user: {
                  select: {
                    nickname: true,
                    avatar: true
                  }
                }
              }
            } : false,
            _count: {
              select: {
                reviews: true,
                favorites: true
              }
            }
          }
        });

        if (!product) return null;

        // 计算平均评分
        if (options.includeReviews && product.reviews) {
          const totalRating = product.reviews.reduce((sum, review) => sum + review.rating, 0);
          (product as any).averageRating = product.reviews.length > 0 
            ? Number((totalRating / product.reviews.length).toFixed(1))
            : 0;
        }

        return product;
      },
      {
        layers: [CacheLayer.REDIS, CacheLayer.NEXT],
        ttl: {
          redis: CacheTTL.LONG,
          next: CacheTTL.MEDIUM
        },
        tags: ['products', `product:${id}`]
      }
    );
  },
  ['product'],
  { 
    revalidate: CacheTTL.MEDIUM,
    tags: ['products']
  }
);

// 获取商品列表 - 动态缓存
export async function getProducts(params: ProductQueryParams): Promise<ProductListResponse> {
  const {
    categoryId,
    searchTerm,
    sortBy = 'default',
    priceRange,
    brands,
    page = 1,
    limit = 20,
    includeFilters = false
  } = params;

  const cacheKey = CacheKeyGenerator.productList(params);
  
  return CacheManager.getOrSet(
    cacheKey,
    async () => {
      // 构建查询条件
      const where: any = {
        status: 'ACTIVE',
        stock: { gt: 0 }
      };

      if (categoryId) {
        where.categoryId = categoryId;
      }

      if (searchTerm) {
        where.OR = [
          { title: { contains: searchTerm, mode: 'insensitive' } },
          { description: { contains: searchTerm, mode: 'insensitive' } },
          { brand: { name: { contains: searchTerm, mode: 'insensitive' } } }
        ];
      }

      if (priceRange) {
        where.price = {
          gte: priceRange[0],
          lte: priceRange[1]
        };
      }

      if (brands && brands.length > 0) {
        where.brandId = { in: brands };
      }

      // 构建排序条件
      const orderBy: any[] = [];
      switch (sortBy) {
        case 'price_asc':
          orderBy.push({ price: 'asc' });
          break;
        case 'price_desc':
          orderBy.push({ price: 'desc' });
          break;
        case 'sales':
          orderBy.push({ sales: 'desc' });
          break;
        case 'newest':
          orderBy.push({ createdAt: 'desc' });
          break;
        default:
          orderBy.push({ sort: 'asc' }, { createdAt: 'desc' });
      }

      // 并行执行查询
      const [products, totalCount, filtersData] = await Promise.all([
        // 商品列表
        prisma.product.findMany({
          where,
          include: {
            brand: true,
            category: true,
            _count: {
              select: {
                reviews: true
              }
            }
          },
          orderBy,
          skip: (page - 1) * limit,
          take: limit
        }),
        
        // 总数统计
        prisma.product.count({ where }),
        
        // 筛选器数据(如果需要)
        includeFilters ? getProductFilters(where) : null
      ]);

      // 计算平均评分
      const productsWithRating = await Promise.all(
        products.map(async (product) => {
          const avgRating = await prisma.review.aggregate({
            where: {
              productId: product.id,
              status: 'APPROVED'
            },
            _avg: {
              rating: true
            }
          });

          return {
            ...product,
            averageRating: avgRating._avg.rating ? Number(avgRating._avg.rating.toFixed(1)) : 0
          };
        })
      );

      return {
        products: productsWithRating,
        pagination: {
          page,
          limit,
          total: totalCount,
          hasNext: page * limit < totalCount,
          hasPrev: page > 1
        },
        filters: filtersData || undefined
      };
    },
    {
      layers: [CacheLayer.REDIS],
      ttl: {
        redis: CacheTTL.SHORT // 商品列表缓存时间较短
      },
      tags: ['products', 'product-list']
    }
  );
}

// 获取筛选器数据
async function getProductFilters(baseWhere: any) {
  const [brands, priceRanges, categories] = await Promise.all([
    // 品牌筛选
    prisma.product.groupBy({
      by: ['brandId'],
      where: baseWhere,
      _count: true,
      orderBy: {
        _count: {
          brandId: 'desc'
        }
      }
    }).then(async (result) => {
      const brandIds = result.map(r => r.brandId).filter(Boolean);
      const brands = await prisma.brand.findMany({
        where: { id: { in: brandIds } },
        select: { id: true, name: true }
      });
      
      return result.map(r => {
        const brand = brands.find(b => b.id === r.brandId);
        return {
          id: r.brandId || '',
          name: brand?.name || '未知品牌',
          count: r._count
        };
      }).filter(b => b.id);
    }),

    // 价格区间筛选
    prisma.product.aggregate({
      where: baseWhere,
      _min: { price: true },
      _max: { price: true }
    }).then(result => {
      const min = result._min.price?.toNumber() || 0;
      const max = result._max.price?.toNumber() || 1000;
      const step = (max - min) / 5;
      
      return Array.from({ length: 5 }, (_, i) => ({
        min: Math.round(min + step * i),
        max: Math.round(min + step * (i + 1)),
        count: 0 // 简化实现，实际需要查询每个区间的商品数量
      }));
    }),

    // 分类筛选
    prisma.product.groupBy({
      by: ['categoryId'],
      where: baseWhere,
      _count: true,
      orderBy: {
        _count: {
          categoryId: 'desc'
        }
      }
    }).then(async (result) => {
      const categoryIds = result.map(r => r.categoryId);
      const categories = await prisma.category.findMany({
        where: { id: { in: categoryIds } },
        select: { id: true, name: true }
      });
      
      return result.map(r => {
        const category = categories.find(c => c.id === r.categoryId);
        return {
          id: r.categoryId,
          name: category?.name || '未知分类',
          count: r._count
        };
      });
    })
  ]);

  return { brands, priceRanges, categories };
}

// 获取相关商品
export const getRelatedProducts = CacheManager.createNextjsCache(
  async (productId: string, options: { limit?: number } = {}): Promise<Product[]> => {
    const { limit = 8 } = options;
    
    // 获取原商品信息
    const product = await prisma.product.findUnique({
      where: { id: productId },
      select: { categoryId: true, brandId: true }
    });

    if (!product) return [];

    // 查找相关商品：同分类或同品牌
    const relatedProducts = await prisma.product.findMany({
      where: {
        AND: [
          { id: { not: productId } },
          { status: 'ACTIVE' },
          { stock: { gt: 0 } },
          {
            OR: [
              { categoryId: product.categoryId },
              { brandId: product.brandId }
            ]
          }
        ]
      },
      include: {
        brand: true,
        _count: {
          select: { reviews: true }
        }
      },
      orderBy: [
        { sales: 'desc' },
        { createdAt: 'desc' }
      ],
      take: limit
    });

    return relatedProducts;
  },
  ['related-products'],
  { 
    revalidate: CacheTTL.MEDIUM,
    tags: ['products', 'related-products']
  }
);

// 获取热销商品
export const getHotProducts = CacheManager.createNextjsCache(
  async (options: { limit?: number; categoryId?: string } = {}): Promise<Product[]> => {
    const { limit = 10, categoryId } = options;
    
    const where: any = {
      status: 'ACTIVE',
      stock: { gt: 0 }
    };

    if (categoryId) {
      where.categoryId = categoryId;
    }

    const hotProducts = await prisma.product.findMany({
      where,
      include: {
        brand: true,
        _count: {
          select: { reviews: true }
        }
      },
      orderBy: [
        { sales: 'desc' },
        { createdAt: 'desc' }
      ],
      take: limit
    });

    return hotProducts;
  },
  ['hot-products'],
  { 
    revalidate: CacheTTL.LONG,
    tags: ['products', 'hot-products']
  }
);

// 搜索商品
export async function searchProducts(
  query: string,
  filters: Omit<ProductQueryParams, 'searchTerm'> = {}
): Promise<ProductListResponse> {
  return getProducts({
    ...filters,
    searchTerm: query
  });
}

// 缓存失效函数
export async function invalidateProductCache(productId?: string) {
  const tags = ['products', 'product-list', 'hot-products', 'related-products'];
  
  if (productId) {
    tags.push(`product:${productId}`);
  }

  await CacheManager.invalidate(tags);
}
```

### 5.3 图片优化和CDN集成

```typescript
// components/ui/OptimizedImage.tsx
'use client';
import Image from 'next/image';
import { useState, useRef, useEffect } from 'react';
import { cn } from '@/lib/utils';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  aspectRatio?: string;
  className?: string;
  priority?: boolean;
  quality?: number;
  placeholder?: 'blur' | 'empty';
  blurDataURL?: string;
  sizes?: string;
  fill?: boolean;
  objectFit?: 'contain' | 'cover' | 'fill' | 'none' | 'scale-down';
  onLoad?: () => void;
  onError?: () => void;
  lazy?: boolean;
  webpFallback?: boolean;
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  aspectRatio,
  className,
  priority = false,
  quality = 80,
  placeholder = 'empty',
  blurDataURL,
  sizes,
  fill = false,
  objectFit = 'cover',
  onLoad,
  onError,
  lazy = true,
  webpFallback = true,
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);
  const [currentSrc, setCurrentSrc] = useState(src);
  const imgRef = useRef<HTMLImageElement>(null);

  // 网络状态检测
  const [connectionSpeed, setConnectionSpeed] = useState<'slow' | 'fast'>('fast');

  useEffect(() => {
    if ('connection' in navigator) {
      const connection = (navigator as any).connection;
      const updateConnectionSpeed = () => {
        const effectiveType = connection.effectiveType;
        setConnectionSpeed(['slow-2g', '2g', '3g'].includes(effectiveType) ? 'slow' : 'fast');
      };
      
      updateConnectionSpeed();
      connection.addEventListener('change', updateConnectionSpeed);
      
      return () => {
        connection.removeEventListener('change', updateConnectionSpeed);
      };
    }
  }, []);

  // 根据网络速度调整质量
  const adjustedQuality = connectionSpeed === 'slow' ? Math.max(50, quality - 20) : quality;

  // 生成响应式图片URL
  const generateSrcSet = (baseSrc: string) => {
    if (!width || !height) return undefined;
    
    const sizes = [0.5, 1, 1.5, 2]; // 不同分辨率倍数
    return sizes.map(scale => {
      const scaledWidth = Math.round(width * scale);
      const scaledHeight = Math.round(height * scale);
      const url = new URL(baseSrc);
      url.searchParams.set('w', scaledWidth.toString());
      url.searchParams.set('h', scaledHeight.toString());
      url.searchParams.set('q', adjustedQuality.toString());
      return `${url.toString()} ${scale}x`;
    }).join(', ');
  };

  // 处理图片加载完成
  const handleLoad = () => {
    setIsLoading(false);
    onLoad?.();
  };

  // 处理图片加载错误
  const handleError = () => {
    setHasError(true);
    setIsLoading(false);
    
    // 尝试降级到原始图片
    if (webpFallback && currentSrc.includes('format=webp')) {
      const fallbackSrc = currentSrc.replace('format=webp', 'format=jpeg');
      setCurrentSrc(fallbackSrc);
      setHasError(false);
      return;
    }
    
    onError?.();
  };

  // 生成blur placeholder
  const generateBlurPlaceholder = () => {
    if (blurDataURL) return blurDataURL;
    
    // 生成简单的颜色placeholder
    const canvas = document.createElement('canvas');
    canvas.width = 1;
    canvas.height = 1;
    const ctx = canvas.getContext('2d');
    if (ctx) {
      ctx.fillStyle = '#f3f4f6';
      ctx.fillRect(0, 0, 1, 1);
      return canvas.toDataURL();
    }
    
    return undefined;
  };

  // 错误状态渲染
  if (hasError) {
    return (
      <div 
        className={cn(
          "flex items-center justify-center bg-gray-100 text-gray-400",
          className
        )}
        style={{ 
          width: fill ? '100%' : width, 
          height: fill ? '100%' : height,
          aspectRatio 
        }}
      >
        <div className="text-center p-4">
          <svg className="w-8 h-8 mx-auto mb-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={1} d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
          </svg>
          <span className="text-xs">图片加载失败</span>
        </div>
      </div>
    );
  }

  const imageProps = {
    ref: imgRef,
    src: currentSrc,
    alt,
    className: cn(
      "transition-all duration-300 ease-in-out",
      isLoading ? "opacity-0 scale-105" : "opacity-100 scale-100",
      className
    ),
    onLoad: handleLoad,
    onError: handleError,
    quality: adjustedQuality,
    priority,
    placeholder: placeholder === 'blur' ? 'blur' as const : 'empty' as const,
    blurDataURL: placeholder === 'blur' ? generateBlurPlaceholder() : undefined,
    sizes,
    ...(fill
      ? { fill: true, style: { objectFit } }
      : { width, height }
    ),
    ...(lazy && !priority && { loading: 'lazy' as const }),
  };

  // 如果是填充模式，需要相对定位的容器
  if (fill) {
    return (
      <div 
        className={cn("relative", className)}
        style={{ aspectRatio }}
      >
        <Image {...imageProps} className={cn(imageProps.className, "object-cover")} />
        
        {/* 加载状态覆盖层 */}
        {isLoading && (
          <div className="absolute inset-0 bg-gray-200 animate-pulse" />
        )}
      </div>
    );
  }

  return (
    <div className="relative inline-block">
      <Image {...imageProps} />
      
      {/* 加载状态覆盖层 */}
      {isLoading && (
        <div 
          className="absolute inset-0 bg-gray-200 animate-pulse"
          style={{ width, height }}
        />
      )}
    </div>
  );
}

// 图片CDN配置
// next.config.js 更新
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    domains: [
      'nsr.itmacode.cn',
      'cdn.shopxo.net',
      'img.shopxo.net',
      'static.shopxo.net'
    ],
    formats: ['image/webp', 'image/avif'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 60 * 60 * 24 * 365, // 1年
    dangerouslyAllowSVG: true,
    contentSecurityPolicy: "default-src 'self'; script-src 'none'; sandbox;",
    
    // 自定义loader用于CDN集成
    loader: 'custom',
    loaderFile: './lib/image-loader.js',
  },
  
  // 静态资源优化
  assetPrefix: process.env.CDN_URL || '',
  
  // 压缩配置
  compress: true,
  
  // 实验性功能
  experimental: {
    optimizeCss: true,
    optimizePackageImports: [
      'lucide-react',
      '@radix-ui/react-icons',
      'framer-motion'
    ],
    turbo: {
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js',
        },
      },
    },
  },
  
  // Webpack配置优化
  webpack: (config, { isServer }) => {
    // 优化bundle大小
    if (!isServer) {
      config.resolve.fallback = {
        ...config.resolve.fallback,
        fs: false,
        net: false,
        tls: false,
      };
    }
    
    // SVG处理
    config.module.rules.push({
      test: /\.svg$/,
      use: ['@svgr/webpack'],
    });
    
    return config;
  },
  
  // Headers优化
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
      {
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=300, s-maxage=600, stale-while-revalidate=86400',
          },
        ],
      },
      {
        source: '/_next/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
      {
        source: '/images/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000',
          },
        ],
      },
    ];
  },
};

module.exports = nextConfig;

// 自定义图片加载器
// lib/image-loader.js
export default function cloudinaryLoader({ src, width, quality }) {
  const params = ['f_auto', 'c_limit'];
  
  if (width) {
    params.push(`w_${width}`);
  }
  
  if (quality) {
    params.push(`q_${quality}`);
  }
  
  const paramsString = params.join(',');
  return `https://cdn.shopxo.net/image/upload/${paramsString}/${src}`;
}
```

### 5.4 性能监控和优化

```typescript
// lib/performance/monitor.ts
export class PerformanceMonitor {
  private static metrics: Map<string, any> = new Map();
  
  // 页面性能监控
  static monitorPagePerformance() {
    if (typeof window === 'undefined') return;
    
    // Core Web Vitals
    this.measureWebVitals();
    
    // 自定义性能指标
    this.measureCustomMetrics();
    
    // 资源加载监控
    this.monitorResourceLoading();
  }
  
  private static measureWebVitals() {
    // Largest Contentful Paint (LCP)
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const lcp = entries[entries.length - 1];
      this.recordMetric('LCP', lcp.startTime);
      
      if (lcp.startTime > 2500) {
        console.warn('LCP is poor:', lcp.startTime);
        this.reportPerformanceIssue('LCP', lcp.startTime);
      }
    }).observe({ entryTypes: ['largest-contentful-paint'] });
    
    // First Input Delay (FID)
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      entries.forEach((entry: any) => {
        const fid = entry.processingStart - entry.startTime;
        this.recordMetric('FID', fid);
        
        if (fid > 100) {
          console.warn('FID is poor:', fid);
          this.reportPerformanceIssue('FID', fid);
        }
      });
    }).observe({ entryTypes: ['first-input'] });
    
    // Cumulative Layout Shift (CLS)
    let clsValue = 0;
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      entries.forEach((entry: any) => {
        if (!entry.hadRecentInput) {
          clsValue += entry.value;
        }
      });
      
      this.recordMetric('CLS', clsValue);
      
      if (clsValue > 0.1) {
        console.warn('CLS is poor:', clsValue);
        this.reportPerformanceIssue('CLS', clsValue);
      }
    }).observe({ entryTypes: ['layout-shift'] });
  }
  
  private static measureCustomMetrics() {
    // 首次内容绘制时间
    const fcpObserver = new PerformanceObserver((list) => {
      const entries = list.getEntries();
      const fcp = entries[0];
      this.recordMetric('FCP', fcp.startTime);
    });
    fcpObserver.observe({ entryTypes: ['paint'] });
    
    // 页面加载完成时间
    window.addEventListener('load', () => {
      const loadTime = performance.timing.loadEventEnd - performance.timing.navigationStart;
      this.recordMetric('PageLoadTime', loadTime);
    });
    
    // DOM交互时间
    document.addEventListener('DOMContentLoaded', () => {
      const domInteractive = performance.timing.domInteractive - performance.timing.navigationStart;
      this.recordMetric('DOMInteractive', domInteractive);
    });
  }
  
  private static monitorResourceLoading() {
    // 监控慢资源
    new PerformanceObserver((list) => {
      const entries = list.getEntries();
      entries.forEach((entry: any) => {
        if (entry.duration > 1000) { // 超过1秒的资源
          console.warn('Slow resource:', entry.name, entry.duration);
          this.reportPerformanceIssue('SlowResource', {
            name: entry.name,
            duration: entry.duration,
            type: entry.initiatorType
          });
        }
      });
    }).observe({ entryTypes: ['resource'] });
  }
  
  private static recordMetric(name: string, value: any) {
    this.metrics.set(name, {
      value,
      timestamp: Date.now(),
      url: window.location.href
    });
  }
  
  private static async reportPerformanceIssue(type: string, data: any) {
    try {
      await fetch('/api/analytics/performance', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          type,
          data,
          userAgent: navigator.userAgent,
          url: window.location.href,
          timestamp: Date.now()
        })
      });
    } catch (error) {
      console.error('Failed to report performance issue:', error);
    }
  }
  
  // 获取性能报告
  static getPerformanceReport() {
    return {
      metrics: Object.fromEntries(this.metrics),
      navigation: performance.getEntriesByType('navigation')[0],
      resources: performance.getEntriesByType('resource').map(r => ({
        name: r.name,
        duration: r.duration,
        size: (r as any).transferSize || 0,
        type: (r as any).initiatorType
      }))
    };
  }
  
  // 内存使用监控
  static monitorMemoryUsage() {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      this.recordMetric('MemoryUsed', memory.usedJSHeapSize);
      this.recordMetric('MemoryTotal', memory.totalJSHeapSize);
      this.recordMetric('MemoryLimit', memory.jsHeapSizeLimit);
      
      // 内存使用率超过80%时警告
      const usageRatio = memory.usedJSHeapSize / memory.jsHeapSizeLimit;
      if (usageRatio > 0.8) {
        console.warn('High memory usage:', usageRatio);
        this.reportPerformanceIssue('HighMemoryUsage', {
          usageRatio,
          used: memory.usedJSHeapSize,
          total: memory.totalJSHeapSize
        });
      }
    }
  }
}

// 性能优化Hook
// hooks/use-performance.ts
import { useEffect, useState } from 'react';

export function usePerformance() {
  const [isOnline, setIsOnline] = useState(true);
  const [networkSpeed, setNetworkSpeed] = useState<'slow' | 'fast'>('fast');
  const [memoryPressure, setMemoryPressure] = useState<'low' | 'medium' | 'high'>('low');

  useEffect(() => {
    // 网络状态监听
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    // 网络速度检测
    if ('connection' in navigator) {
      const connection = (navigator as any).connection;
      const updateNetworkSpeed = () => {
        const effectiveType = connection.effectiveType;
        setNetworkSpeed(['slow-2g', '2g', '3g'].includes(effectiveType) ? 'slow' : 'fast');
      };
      
      updateNetworkSpeed();
      connection.addEventListener('change', updateNetworkSpeed);
    }

    // 内存压力监控
    const monitorMemory = () => {
      if ('memory' in performance) {
        const memory = (performance as any).memory;
        const usageRatio = memory.usedJSHeapSize / memory.jsHeapSizeLimit;
        
        if (usageRatio > 0.8) {
          setMemoryPressure('high');
        } else if (usageRatio > 0.6) {
          setMemoryPressure('medium');
        } else {
          setMemoryPressure('low');
        }
      }
    };

    monitorMemory();
    const memoryInterval = setInterval(monitorMemory, 30000); // 每30秒检查一次

    // 启动性能监控
    PerformanceMonitor.monitorPagePerformance();

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
      clearInterval(memoryInterval);
    };
  }, []);

  return {
    isOnline,
    networkSpeed,
    memoryPressure,
    shouldUseOptimizedAssets: networkSpeed === 'slow' || memoryPressure === 'high'
  };
}

// 组件级性能优化
// components/ui/LazyComponent.tsx
import { lazy, Suspense, ComponentType } from 'react';
import { usePerformance } from '@/hooks/use-performance';

interface LazyComponentProps {
  factory: () => Promise<{ default: ComponentType<any> }>;
  fallback?: React.ReactNode;
  props?: any;
  condition?: boolean;
}

export function LazyComponent({ 
  factory, 
  fallback = <div className="animate-pulse bg-gray-200 h-32 rounded" />, 
  props = {},
  condition = true 
}: LazyComponentProps) {
  const { memoryPressure, networkSpeed } = usePerformance();
  
  // 在低内存或慢网络情况下，不加载非关键组件
  if (!condition || (memoryPressure === 'high' && networkSpeed === 'slow')) {
    return <>{fallback}</>;
  }
  
  const LazyLoadedComponent = lazy(factory);
  
  return (
    <Suspense fallback={fallback}>
      <LazyLoadedComponent {...props} />
    </Suspense>
  );
}

// 使用示例
export function ProductDetailPage() {
  return (
    <div>
      {/* 关键内容立即渲染 */}
      <ProductInfo />
      <AddToCartSection />
      
      {/* 非关键内容懒加载 */}
      <LazyComponent
        factory={() => import('./RelatedProducts')}
        fallback={<div className="h-64 bg-gray-100 animate-pulse rounded" />}
      />
      
      <LazyComponent
        factory={() => import('./ProductReviews')}
        fallback={<div className="h-48 bg-gray-100 animate-pulse rounded" />}
        condition={true} // 总是加载评论，因为对用户决策很重要
      />
    </div>
  );
}
```

---

继续完善文档内容...