# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ShopXO UniApp v6.6.0 - A cross-platform e-commerce mobile application built with UniApp framework supporting mini-programs (WeChat, Alipay, QQ, Baidu, ByteDance, Kuaishou), H5, and native mobile apps. The project features a comprehensive plugin ecosystem, visual DIY page builder, and multi-tenant architecture.

## Development Environment

- **Primary IDE**: HBuilderX (recommended for UniApp development)
- **Framework**: UniApp with Vue 2.x
- **Language Support**: Vue I18n (Chinese/English)
- **Build Tool**: Vue CLI with custom webpack configuration

## Key Configuration Files

- `App.vue` - Global configuration, API endpoints, theme settings
- `manifest.json` - Platform-specific configurations and app metadata
- `pages.json` - Page routing and tabbar configuration
- `vue.config.js` - Webpack performance optimization
- `locale/index.js` - Multi-language configuration

## Essential Setup Steps

1. **Backend Requirements**: Requires ShopXO backend system installation
2. **API Configuration**: Update `App.vue` with your backend URLs:
   - `request_url`: API endpoint base URL
   - `static_url`: Static resources URL (append `/public/` if needed)
3. **Theme Selection**: Modify `default_theme` in App.vue (8 themes available: red, yellow, black, green, orange, blue, brown, purple)
4. **Language Setup**: Configure `default_language` in App.vue (zh/en supported)

## Project Architecture

### Core Directory Structure

- `pages/` - All application pages including plugins and DIY components
- `components/` - 60+ reusable global components
- `common/` - Shared JavaScript utilities and global styles
- `static/images/` - Theme-based image assets organized by color
- `uni_modules/` - UniApp ecosystem modules and third-party components
- `locale/` - Multi-language translation files

### Plugin System (30+ Plugins)

Located in `pages/plugins/`, each plugin is self-contained:
- **E-commerce**: `seckill`, `coupon`, `distribution`, `wallet`
- **Content**: `blog`, `ask`, `article`
- **Store Management**: `shop`, `realstore`
- **Member System**: `membershiplevelvip`, `signin`, `points`
- **Payment**: `coin`, `invoice`, `scanpay`
- **Logistics**: `delivery`, `express`

### DIY Visual Builder (40+ Components)

Located in `pages/diy/components/diy/`, supports:
- Layout components: `carousel`, `tabs`, `header`
- Content components: `rich-text`, `video`, `img-magic`
- E-commerce components: `goods-list`, `goods-magic`, `seckill`
- Data components: `data-magic`, `search`, `notice`

## Development Commands

Since this is a UniApp project without package.json scripts, use HBuilderX for development:

### Running Development Server
```bash
# Open project in HBuilderX
# Use: 运行 -> 运行到小程序模拟器 -> [选择平台]
# Or: 运行 -> 运行到浏览器 -> [选择浏览器]
```

### Building for Production
```bash
# Use HBuilderX: 发行 -> [选择目标平台]
# Platforms: 微信小程序、H5、App(iOS/Android)、其他小程序平台
```

## Component Development Patterns

### Global Components
- Located in `/components/[component-name]/[component-name].vue`
- Follow existing naming conventions (kebab-case)
- Include component-specific CSS files when needed

### Plugin Components
- Self-contained within plugin directories
- Use plugin-specific styling and logic
- Follow the pattern: `pages/plugins/[plugin-name]/[page-name]/[page-name].vue`

### DIY Components
- Located in `pages/diy/components/diy/`
- Support data binding and configuration
- Include modular sub-components in `modules/` directory

## Styling Guidelines

### Theme System
- 8 predefined color themes with corresponding image assets
- Theme colors defined in `App.vue` global data
- Images organized by theme in `static/images/[theme-color]/`

### CSS Architecture
- Global styles in `common/css/`
- Component-specific styles co-located with components
- SCSS support via `uni_modules/uni-scss`

## Multi-language Implementation

### Adding New Languages
1. Add language object to `App.vue` globalData.language_list
2. Create translation file in `locale/[lang-code].json`
3. Update `locale/index.js` imports
4. Use `$t('key')` in templates for translations

### Translation Keys
- Organized by feature/page in JSON files
- Use nested objects for better organization
- Follow existing key naming patterns

## API Integration

### Base Configuration
- API base URL configured in `App.vue` globalData.request_url
- Static resources URL in globalData.static_url
- Request handling in `common/js/common/common.js`

### Backend Dependencies
- Requires ShopXO PHP backend system
- API documentation: https://doc.shopxo.net/article/2.html
- Authentication tokens handled automatically

## Platform-Specific Considerations

### Mini-Program Development
- WeChat AppID: `wxd7c94eeac9cb17f2`
- Alipay AppID: `2021001173639600`
- Platform-specific configurations in `manifest.json`

### H5 Development
- Custom template in `template.h5.html`
- Router mode: hash
- Development port: 8082

### App Development
- Native module integrations: Payment, OAuth, Maps, Camera
- Platform permissions configured in `manifest.json`
- Splash screen and launch configurations

## Key Features to Understand

1. **Visual DIY System**: Drag-and-drop page builder with extensive component library
2. **Plugin Architecture**: Modular business functionality with isolated components
3. **Multi-tenant Support**: Shop and realstore plugins for multi-merchant scenarios
4. **Theme System**: Dynamic color theming with asset management
5. **Cross-platform Compatibility**: Single codebase targeting 7+ platforms
6. **Backend Integration**: Deep integration with ShopXO e-commerce backend

## Common Customization Points

- Theme colors and branding in `App.vue`
- Navigation structure in `pages.json`
- Component styling in respective CSS files
- Plugin functionality in plugin-specific directories
- DIY component library in `pages/diy/components/diy/`