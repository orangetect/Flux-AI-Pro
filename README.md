# ============================================
# Cloudflare Workers 配置文件
# 項目: Flux AI Pro v9.6.0-gen-api
# 更新: 2025-12-17
# ============================================

name = "flux-ai-pro"
main = "worker.js"
compatibility_date = "2024-12-01"

# ============================================
# Workers AI 綁定（用於中文自動翻譯）
# ============================================
[ai]
binding = "AI"

# ============================================
# 環境變量（非敏感）
# ============================================
[vars]
WORKER_VERSION = "9.6.0-gen-api"
DEFAULT_MODEL = "flux"
MAX_WIDTH = 2048
MAX_HEIGHT = 2048
MAX_TIMEOUT = 120000
ENVIRONMENT = "production"
API_ENDPOINT = "https://gen.pollinations.ai"

# ============================================
# 自定義域名（可選 - 如不需要請註釋掉）
# ============================================
# routes = [
#   { pattern = "flux-ai.your-domain.com/*", zone_name = "your-domain.com" }
# ]

# ============================================
# Secrets 通過 wrangler secret 命令設置
# 🔐 必需設置 POLLINATIONS_API_KEY
# ============================================
# 設置命令：
# wrangler secret put POLLINATIONS_API_KEY
這就是完整的 wrangler.toml 文件，已經：

✅ 移除了 [limits] 區塊（Free Plan 不支援）
✅ 保留了 Workers AI 綁定（用於中文翻譯）
✅ 設置了必要的環境變量
✅ 配置了新的 API 端點 https://gen.pollinations.ai

部署步驟
bash
# 1. 保存上面的 wrangler.toml 到項目根目錄

# 2. 設置 API Key
wrangler secret put POLLINATIONS_API_KEY
# 輸入你的 Pollinations API Key

# 3. 部署
wrangler deploy

# 4. 訪問你的 Worker
# https://flux-ai-pro.your-account.workers.dev
現在應該可以成功部署了！

KV空間設置
KV 命名空間完整設置指南
1. 創建 KV 命名空間
bash
# 創建生產環境 KV
wrangler kv:namespace create "FLUX_CACHE"

# 輸出示例：
# 🌀 Creating namespace with title "flux-ai-pro-FLUX_CACHE"
# ✨ Success!
# Add the following to your wrangler.toml:
# [[kv_namespaces]]
# binding = "FLUX_CACHE"
# id = "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"

# 創建預覽環境 KV（用於本地開發）
wrangler kv:namespace create "FLUX_CACHE" --preview

# 輸出示例：
# ✨ Success!
# Add the following to your wrangler.toml:
# preview_id = "x9y8z7w6v5u4t3s2r1q0p9o8n7m6l5k4"
2. 更新 wrangler.toml
將上面命令返回的 ID 添加到配置文件：

text
# ============================================
# Cloudflare Workers 配置文件
# 項目: Flux AI Pro v9.6.0-gen-api
# 更新: 2025-12-17
# ============================================

name = "flux-ai-pro"
main = "worker.js"
compatibility_date = "2024-12-01"

# ============================================
# Workers AI 綁定（用於中文自動翻譯）
# ============================================
[ai]
binding = "AI"

# ============================================
# 環境變量（非敏感）
# ============================================
[vars]
WORKER_VERSION = "9.6.0-gen-api"
DEFAULT_MODEL = "flux"
MAX_WIDTH = 2048
MAX_HEIGHT = 2048
MAX_TIMEOUT = 120000
ENVIRONMENT = "production"
API_ENDPOINT = "https://gen.pollinations.ai"

# ============================================
# KV 命名空間（圖片緩存和速率限制）
# ✅ 替換下面的 ID 為你實際創建的 ID
# ============================================
[[kv_namespaces]]
binding = "FLUX_CACHE"
id = "替換為你的_kv_namespace_id"
preview_id = "替換為你的_preview_kv_namespace_id"

# ============================================
# 自定義域名（可選）
# ============================================
# routes = [
#   { pattern = "flux-ai.your-domain.com/*", zone_name = "your-domain.com" }
# ]

# ============================================
# Secrets 設置
# ============================================
# wrangler secret put POLLINATIONS_API_KEY
3. 在 worker.js 中使用 KV
在 worker.js 的主入口函數中添加 KV 緩存邏輯：

javascript
// 在 export default { async fetch(request, env, ctx) { 的開頭添加

export default {
    async fetch(request, env, ctx) {
        const url = new URL(request.url);
        const startTime = Date.now();
        const clientIP = getClientIP(request);
        
        // ✅ KV 緩存檢查（用於圖片緩存）
        if (url.pathname === '/_internal/generate' && request.method === 'POST') {
            const body = await request.clone().json();
            const cacheKey = generateCacheKey(body);
            
            // 檢查緩存
            if (env.FLUX_CACHE) {
                const cached = await env.FLUX_CACHE.get(cacheKey, { type: 'arrayBuffer' });
                if (cached) {
                    console.log('✅ Cache hit:', cacheKey);
                    return new Response(cached, {
                        headers: {
                            'Content-Type': 'image/png',
                            'X-Cache': 'HIT',
                            'X-Cache-Key': cacheKey,
                            ...corsHeaders()
                        }
                    });
                }
            }
        }
        
        // 原有代碼繼續...
        // ...
    }
};

// ✅ 生成緩存鍵函數（在文件末尾添加）
function generateCacheKey(params) {
    const { prompt, model, width, height, seed, style, quality_mode } = params;
    // 只有固定 seed 才緩存（seed !== -1）
    if (seed === -1 || seed === undefined) return null;
    
    const keyData = {
        prompt: prompt.toLowerCase().trim(),
        model: model || 'zimage',
        width: width || 1024,
        height: height || 1024,
        seed: seed,
        style: style || 'none',
        quality_mode: quality_mode || 'standard'
    };
    
    // 使用 JSON 字符串生成簡單哈希
    const keyString = JSON.stringify(keyData);
    return 'img_' + btoa(keyString).substring(0, 32);
}
4. 更新 handleInternalGenerate 函數保存緩存
在 handleInternalGenerate 函數的成功返回前添加：

javascript
async function handleInternalGenerate(request, env, ctx) {
    const logger = new Logger();
    const startTime = Date.now();
    
    try {
        const body = await request.json();
        // ... 原有代碼 ...
        
        const router = new MultiProviderRouter({}, env);
        const results = await router.generate(prompt, options, logger);
        
        const duration = Date.now() - startTime;
        
        // ✅ 如果只生成一張圖片
        if (results.length === 1 && results[0].imageData) {
            const result = results[0];
            
            // ✅ 保存到 KV 緩存（僅固定 seed）
            if (env.FLUX_CACHE && options.seed !== -1) {
                const cacheKey = generateCacheKey(body);
                if (cacheKey) {
                    // 緩存 7 天（604800 秒）
                    await env.FLUX_CACHE.put(cacheKey, result.imageData, {
                        expirationTtl: 604800,
                        metadata: {
                            model: result.model,
                            seed: result.seed,
                            width: result.width,
                            height: result.height,
                            created: new Date().toISOString()
                        }
                    });
                    console.log('✅ Cached:', cacheKey);
                }
            }
            
            return new Response(result.imageData, {
                headers: {
                    'Content-Type': result.contentType || 'image/png',
                    'X-Cache': 'MISS',
                    // ... 其他 headers
                }
            });
        }
        // ... 原有代碼繼續 ...
    } catch (e) {
        // ... 錯誤處理 ...
    }
}
5. KV 速率限制實現（可選）
javascript
// 在 export default { async fetch() 開頭添加速率限制

async function checkRateLimit(env, clientIP) {
    if (!env.FLUX_CACHE) return { allowed: true };
    
    const rateLimitKey = `ratelimit_${clientIP}`;
    const current = await env.FLUX_CACHE.get(rateLimitKey);
    
    const limit = 10; // 每分鐘 10 次請求
    const window = 60; // 60 秒窗口
    
    if (current) {
        const count = parseInt(current);
        if (count >= limit) {
            return { 
                allowed: false, 
                remaining: 0, 
                reset: window 
            };
        }
        // 增加計數
        await env.FLUX_CACHE.put(rateLimitKey, (count + 1).toString(), { expirationTtl: window });
        return { 
            allowed: true, 
            remaining: limit - count - 1, 
            reset: window 
        };
    } else {
        // 首次請求
        await env.FLUX_CACHE.put(rateLimitKey, "1", { expirationTtl: window });
        return { 
            allowed: true, 
            remaining: limit - 1, 
            reset: window 
        };
    }
}

// 在主入口使用
export default {
    async fetch(request, env, ctx) {
        const clientIP = getClientIP(request);
        
        // ✅ 速率限制檢查
        if (url.pathname === '/_internal/generate') {
            const rateLimit = await checkRateLimit(env, clientIP);
            if (!rateLimit.allowed) {
                return new Response(JSON.stringify({
                    error: {
                        message: 'Rate limit exceeded',
                        remaining: 0,
                        reset: rateLimit.reset
                    }
                }), {
                    status: 429,
                    headers: {
                        'Content-Type': 'application/json',
                        'X-RateLimit-Limit': '10',
                        'X-RateLimit-Remaining': '0',
                        'X-RateLimit-Reset': rateLimit.reset.toString(),
                        ...corsHeaders()
                    }
                });
            }
        }
        
        // 原有代碼繼續...
    }
};
6. KV 管理命令
bash
# 查看 KV 命名空間列表
wrangler kv:namespace list

# 列出 KV 中的所有鍵
wrangler kv:key list --namespace-id=你的_kv_id

# 獲取特定鍵的值
wrangler kv:key get "img_xxx" --namespace-id=你的_kv_id

# 刪除特定鍵
wrangler kv:key delete "img_xxx" --namespace-id=你的_kv_id

# 清空整個 KV（謹慎使用）
wrangler kv:key list --namespace-id=你的_kv_id | jq -r '.[] | .name' | xargs -I {} wrangler kv:key delete {} --namespace-id=你的_kv_id
7. Free Plan KV 限制
資源	Free Plan 限制
讀取	100,000/天
寫入	1,000/天
刪除	1,000/天
存儲空間	1 GB
鍵大小	512 bytes
值大小	25 MB
對於圖片緩存來說，寫入限制（1,000/天） 是主要瓶頸，建議：

只緩存固定 seed 的圖片

設置較長的 TTL（7 天）減少重複寫入

優先緩存熱門配置

8. 部署
bash
# 部署帶 KV 的 Worker
wrangler deploy

# 測試緩存
curl -X POST https://your-worker.workers.dev/_internal/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "test cache",
    "seed": 12345,
    "model": "flux"
  }' \
  -I  # 查看 X-Cache header

# 第一次應該返回 X-Cache: MISS
# 第二次相同請求應該返回 X-Cache: HIT
這樣就完成了 KV 緩存的完整設置！

更新介紹README.md
更新後的完整 README.md
text
# 🎨 Flux AI Pro

<div align="center">

![Version](https://img.shields.io/badge/version-9.6.0--gen--api-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Cloudflare](https://img.shields.io/badge/Cloudflare-Workers-orange)
![API](https://img.shields.io/badge/API-gen.pollinations.ai-purple)

**基於 Cloudflare Workers 的專業 AI 圖像生成服務**

**✨ 最新版本：9.6.0-gen-api - 新 API 端點 + 必需認證**

[English](#english) | [中文](#中文) | [更新記錄](#更新記錄)

</div>

---

## 中文

### ✨ 功能特點

- 🎨 **4 個官方模型**
  - **Z-Image Turbo** ⚡ - 6B 參數，極速生成
  - **Flux 標準版** - 平衡速度與質量
  - **Flux Turbo** ⚡ - 超快速生成
  - **Kontext** 🎨 - 支持圖生圖

- 🌐 **完整的 Web UI 界面**
  - 三欄式佈局（參數 | 結果 | 提示詞）
  - 實時配置預覽
  - 歷史記錄管理
  - 響應式設計

- 🚀 **智能優化**
  - 自動中文翻譯（支持 Cloudflare Workers AI）
  - HD 高清增強（3 種質量模式）
  - 智能參數優化
  - 複雜度分析

- 🎯 **風格預設**
  - 8+ 種藝術風格
  - 自定義風格組合
  - 正面/負面提示詞

- 🖼️ **圖生圖支持**
  - Kontext 模型支持參考圖像
  - 多張圖片輸入
  - URL 方式上傳

- 💾 **本地歷史記錄**
  - 自動保存生成記錄
  - 重用參數功能
  - 導出/清空記錄
  - 最多保存 100 條

- 🔐 **API Key 認證** 🆕
  - **必需**：使用官方 API Key
  - Bearer Token 認證方式
  - 環境變量安全配置
  - 支持匿名測試（可能失敗）

- 🆕 **圖片字節返回**
  - 直接返回圖片二進制數據
  - 不暴露上游 API URL
  - 支持單張/批量生成
  - HTTP Headers 傳遞元數據

- 💾 **KV 緩存支持** (可選)
  - 圖片緩存（固定 seed）
  - 速率限制保護
  - 7 天自動過期

### 📦 技術棧

- **運行環境**: Cloudflare Workers
- **AI 翻譯**: Cloudflare Workers AI
- **前端**: 原生 HTML/CSS/JavaScript
- **API**: Pollinations.ai Official API (gen.pollinations.ai)
- **存儲**: localStorage（客戶端）+ KV（可選）

### 🚀 快速開始

#### 1. 克隆項目

git clone https://github.com/kinai9661/Flux-AI-Pro.git
cd Flux-AI-Pro

text

#### 2. 安裝 Wrangler

npm install -g wrangler

text

#### 3. 登錄 Cloudflare

wrangler login

text

#### 4. 配置 API Key（必需） 🔐

wrangler secret put POLLINATIONS_API_KEY

輸入你的 API Key（例如：pk_xxxxxxxxxxxxx）
text

**獲取 API Key：**
- 訪問 [https://pollinations.ai](https://pollinations.ai) 或 [https://enter.pollinations.ai](https://enter.pollinations.ai)
- 註冊並獲取 API Key
- 格式：`pk_xxxxxxxxxxxxx`

#### 5. 部署

wrangler deploy

text

#### 6. 訪問

部署成功後，訪問你的 Worker URL：
https://flux-ai-pro.your-account.workers.dev

text

### ⚙️ 配置說明

#### wrangler.toml 基本配置

name = "flux-ai-pro"
main = "worker.js"
compatibility_date = "2024-12-01"

[ai]
binding = "AI"

[vars]
WORKER_VERSION = "9.6.0-gen-api"
API_ENDPOINT = "https://gen.pollinations.ai"

text

#### 環境變量

| 變量名 | 說明 | 必需 |
|--------|------|------|
| `POLLINATIONS_API_KEY` | Pollinations.ai API Key | ✅ 是 |
| `AI` | Workers AI 綁定（翻譯） | ❌ 可選 |

#### KV 命名空間（可選）

創建 KV 命名空間
wrangler kv:namespace create "FLUX_CACHE"
wrangler kv:namespace create "FLUX_CACHE" --preview

在 wrangler.toml 中配置
[[kv_namespaces]]
binding = "FLUX_CACHE"
id = "your-kv-namespace-id"
preview_id = "your-preview-kv-namespace-id"

text

### 🎨 使用方法

#### 1. 基本生成

1. 輸入提示詞（支持中文）
2. 選擇模型和尺寸
3. 選擇風格（可選）
4. 點擊「開始生成」

#### 2. 進階選項

- **Seed**: 設置隨機種子（-1 為隨機）
- **生成數量**: 1-4 張
- **自動優化**: 智能調整參數
- **HD 增強**: 自動提升質量

#### 3. 圖生圖（Kontext）

1. 選擇 Kontext 模型
2. 在「參考圖像 URL」中輸入圖片地址
3. 輸入提示詞描述變化
4. 生成

#### 4. 歷史記錄

- 自動保存所有生成記錄
- 點擊「重用」快速復用參數
- 點擊圖片查看大圖
- 下載圖片到本地

### 📊 模型對比

| 模型 | 速度 | 質量 | 參數量 | 價格* | 特點 |
|------|------|------|--------|-------|------|
| Z-Image Turbo | ⚡⚡⚡ | ⭐⭐⭐ | 6B | 0.0002 | 極速生成 |
| Flux 標準版 | ⚡⚡ | ⭐⭐⭐⭐ | - | 0.00012 | 平衡 |
| Flux Turbo | ⚡⚡⚡ | ⭐⭐⭐ | - | 0.0003 | 快速 |
| Kontext | ⚡ | ⭐⭐⭐⭐⭐ | - | 0.04 | 圖生圖 |

*價格單位: Pollen credits

### 🎯 質量模式

| 模式 | 最小分辨率 | Steps 倍率 | 適用場景 |
|------|-----------|-----------|---------|
| 經濟模式 | 1024px | 0.85x | 快速預覽 |
| 標準模式 | 1280px | 1.0x | 日常使用 |
| 超高清模式 | 1536px | 1.35x | 高質量輸出 |

### 🎨 內置風格

- 動漫風格 ✨
- 寫實照片 📷
- 油畫 🎨
- 水彩畫 💧
- 賽博朋克 🌃
- 奇幻風格 🐉
- 吉卜力風格 🍃

### 📐 尺寸預設

- 方形: 1024x1024, 1536x1536, 2048x2048
- 豎屏: 1080x1920 (9:16)
- 橫屏: 1920x1080 (16:9)
- Instagram: 1080x1080
- 桌布: 1920x1080 (Full HD)

### 🔧 開發

#### 本地開發

啟動開發服務器
wrangler dev

查看日誌
wrangler tail

查看部署列表
wrangler deployments list

text

#### 文件結構

flux-ai-pro/
├── worker.js # 主程序
├── wrangler.toml # Cloudflare 配置
├── README.md # 說明文檔
├── CHANGELOG.md # 更新記錄
└── package.json # 依賴配置（可選）

text

### 🐛 常見問題

#### Q: 圖片生成失敗？
A: 
1. 檢查是否已設置 `POLLINATIONS_API_KEY`
2. 確認 API Key 有效且有餘額
3. 查看日誌：`wrangler tail`
4. 嘗試更換模型

#### Q: 401/403 認證錯誤？
A: 
重新設置 API Key
wrangler secret delete POLLINATIONS_API_KEY
wrangler secret put POLLINATIONS_API_KEY
wrangler deploy

text

#### Q: 中文翻譯不工作？
A: 確保 `wrangler.toml` 中已綁定 Workers AI：
[ai]
binding = "AI"

text

#### Q: 歷史記錄丟失？
A: 歷史記錄保存在瀏覽器 localStorage，清除瀏覽器數據會丟失。可考慮使用 KV 存儲。

#### Q: 免費計劃 CPU 限制錯誤？
A: 移除 `wrangler.toml` 中的 `[limits]` 配置段（Free Plan 不支持）。

#### Q: 如何啟用 KV 緩存？
A: 
wrangler kv:namespace create "FLUX_CACHE"

將返回的 ID 添加到 wrangler.toml
text

### 📜 更新記錄

#### v9.6.0-gen-api (2025-12-17) 🎉 最新
**🔥 重大更新 - API 端點遷移**

**API 變更**
- ✅ **新端點**: `https://image.pollinations.ai` → `https://gen.pollinations.ai`
- ✅ **URL 格式**: `/prompt/{text}` → `/image/{text}`
- ✅ **必需認證**: 加入 `Authorization: Bearer <token>` header
- ✅ **錯誤處理**: 專門處理 401/403 認證失敗

**認證系統**
- ✅ **API Key 必需**: 從環境變量 `POLLINATIONS_API_KEY` 讀取
- ✅ **Bearer Token**: 標準 OAuth 2.0 認證方式
- ✅ **狀態顯示**: UI 實時顯示認證狀態和 API 端點
- ✅ **警告提示**: 未設置 API Key 時顯示警告

**UI 改進**
- ✅ 頂部導航欄顯示 API 端點地址
- ✅ 認證狀態實時更新（🔐 已認證 / ⚠️ 需要 API Key）
- ✅ 生成結果顯示 API 端點信息
- ✅ 錯誤提示包含解決方案（如何設置 API Key）

**配置更新**
- ✅ `wrangler.toml` 移除 `[limits]`（Free Plan 不支持）
- ✅ 新增 `API_ENDPOINT` 環境變量
- ✅ `CONFIG.POLLINATIONS_AUTH.enabled` 默認為 `true`
- ✅ 自動從 `env.POLLINATIONS_API_KEY` 讀取密鑰

**日誌增強**
- ✅ 請求日誌包含認證狀態
- ✅ API 端點記錄
- ✅ Token 前綴顯示（安全性）
- ✅ 詳細錯誤信息

**破壞性變更** ⚠️
- ❌ **必需 API Key**: 未設置將無法正常使用
- ❌ **舊版 URL 失效**: 舊的 `image.pollinations.ai` 端點可能不再可用
- ❌ **匿名模式受限**: 建議使用 API Key 認證

**遷移指南**
1. 獲取 API Key
訪問 https://pollinations.ai 註冊

2. 設置環境變量
wrangler secret put POLLINATIONS_API_KEY

3. 重新部署
wrangler deploy

text

---

#### v9.5.2-bytes (2025-12-17)
**🎯 圖片字節返回**
- ✅ **返回圖片字節數據**：不再返回 Pollinations URL，直接返回圖片二進制數據
- ✅ **雙響應模式**：
  - 單張圖片：直接返回圖片字節（Content-Type: image/png）
  - 多張圖片：返回 JSON 格式（包含 base64 編碼）

**🔧 技術改進**
- 添加圖片 Blob 和 ArrayBuffer 處理
- HTTP Headers 傳遞元數據（模型、種子、尺寸等）
- Base64 編碼支持批量生成
- ObjectURL 本地緩存優化

**📦 新增功能**
- 圖片字節流式傳輸
- 自定義文件名下載
- Content-Disposition 頭部設置
- 前端 Blob 轉換處理

**🐛 修復問題**
- 修復歷史記錄存儲 Blob URL
- 優化內存釋放機制
- 改進錯誤處理邏輯

---

#### v9.5.1-fixed (2025-12-17)
**✅ Bug 修復**
- ✅ 修復 CSP 內聯事件錯誤（移除所有 `onclick` 等內聯事件）
- ✅ 添加 Favicon（避免 404 錯誤）
- ✅ 修復生成結果不顯示問題
- ✅ 優化歷史記錄顯示邏輯

**🎨 UI 改進**
- 改進生成結果展示動畫
- 添加"剛剛生成"標籤
- 優化成功提示樣式
- 改進錯誤提示顯示

**🔧 代碼優化**
- 移除所有內聯 JavaScript
- 改進事件監聽器綁定
- 優化 CSS 樣式結構
- 清理冗餘代碼

---

#### v9.5.0 (2025-12-16)
**🎉 首次發布**

**核心功能**
- 支持 4 個官方模型（Z-Image Turbo、Flux、Flux Turbo、Kontext）
- 完整的 Web UI 界面（三欄式佈局）
- 智能參數優化系統
- HD 高清增強功能
- 8+ 種藝術風格預設

**多語言支持**
- 自動中文翻譯（基於 Cloudflare Workers AI）
- 中英雙語界面
- 支持中文提示詞輸入

**圖像處理**
- 文生圖功能
- 圖生圖功能（Kontext 模型）
- 多張參考圖像支持
- 自定義尺寸預設

**本地存儲**
- 歷史記錄自動保存（localStorage）
- 最多保存 100 條記錄
- 導出/導入功能
- 一鍵清空

**進階功能**
- 自動參數優化
- 複雜度分析
- 質量模式選擇（經濟/標準/超高清）
- Seed 控制
- 批量生成（1-4 張）

---

### 🔮 未來計劃

#### v10.0 (計劃中)
- [ ] 更多模型支持（SDXL、DALL-E 3）
- [ ] 批量下載功能
- [ ] 圖片編輯工具
- [ ] 自定義風格訓練
- [ ] 團隊協作功能
- [ ] KV 緩存優化
- [ ] R2 持久化存儲

#### v10.1 (規劃中)
- [ ] 視頻生成支持
- [ ] 3D 模型生成
- [ ] AI 圖片放大
- [ ] 移動端優化
- [ ] PWA 支持

#### v11.0 (遠期)
- [ ] API 接口開放
- [ ] 插件系統
- [ ] 雲端同步
- [ ] 多用戶支持
- [ ] 訂閱計劃

---

### 📊 版本對比

| 版本 | 發布日期 | 主要特性 | 破壞性更改 |
|------|---------|---------|-----------|
| 9.6.0-gen-api | 2025-12-17 | 新 API 端點 + 必需認證 | ✅ 是 |
| 9.5.2-bytes | 2025-12-17 | 返回圖片字節 | ✅ 是 |
| 9.5.1-fixed | 2025-12-17 | 修復 CSP 錯誤 | ❌ 否 |
| 9.5.0 | 2025-12-16 | 首次發布 | - |

---

### 📄 許可證

MIT License

### 🤝 貢獻

歡迎提交 Issue 和 Pull Request！

**貢獻指南**
1. Fork 本項目
2. 創建 feature 分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 開啟 Pull Request

### 🔗 相關鏈接

- [Pollinations.ai](https://pollinations.ai/)
- [Pollinations API Docs](https://enter.pollinations.ai/api/docs)
- [Cloudflare Workers](https://workers.cloudflare.com/)
- [Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/)
- [項目主頁](https://github.com/kinai9661/Flux-AI-Pro)
- [問題反饋](https://github.com/kinai9661/Flux-AI-Pro/issues)

---

## English

### ✨ Features

- 🎨 **4 Official Models**
  - **Z-Image Turbo** ⚡ - 6B parameters, lightning fast
  - **Flux Standard** - Balanced speed and quality
  - **Flux Turbo** ⚡ - Ultra-fast generation
  - **Kontext** 🎨 - Supports image-to-image

- 🌐 **Complete Web UI**
  - Three-column layout (Parameters | Results | Prompts)
  - Real-time configuration preview
  - History management
  - Responsive design

- 🚀 **Smart Optimization**
  - Auto Chinese translation (powered by Cloudflare Workers AI)
  - HD enhancement (3 quality modes)
  - Intelligent parameter optimization
  - Complexity analysis

- 🎯 **Style Presets**
  - 8+ artistic styles
  - Custom style combinations
  - Positive/negative prompts

- 🖼️ **Image-to-Image Support**
  - Kontext model supports reference images
  - Multiple image inputs
  - URL-based upload

- 💾 **Local History**
  - Auto-save generation records
  - Reuse parameters feature
  - Export/clear records
  - Up to 100 records

- 🔐 **API Key Authentication** 🆕
  - **Required**: Official API Key
  - Bearer Token authentication
  - Environment variable configuration
  - Anonymous testing (may fail)

- 🆕 **Image Bytes Return**
  - Direct binary data response
  - No exposed upstream API URL
  - Single/batch generation support
  - Metadata via HTTP Headers

- 💾 **KV Cache Support** (Optional)
  - Image caching (fixed seed)
  - Rate limiting protection
  - 7-day auto expiration

### 📦 Tech Stack

- **Runtime**: Cloudflare Workers
- **AI Translation**: Cloudflare Workers AI
- **Frontend**: Native HTML/CSS/JavaScript
- **API**: Pollinations.ai Official API (gen.pollinations.ai)
- **Storage**: localStorage (client-side) + KV (optional)

### 🚀 Quick Start

#### 1. Clone Repository

git clone https://github.com/kinai9661/Flux-AI-Pro.git
cd Flux-AI-Pro

text

#### 2. Install Wrangler

npm install -g wrangler

text

#### 3. Login to Cloudflare

wrangler login

text

#### 4. Configure API Key (Required) 🔐

wrangler secret put POLLINATIONS_API_KEY

Enter your API Key (e.g., pk_xxxxxxxxxxxxx)
text

**Get API Key:**
- Visit [https://pollinations.ai](https://pollinations.ai) or [https://enter.pollinations.ai](https://enter.pollinations.ai)
- Sign up and get API Key
- Format: `pk_xxxxxxxxxxxxx`

#### 5. Deploy

wrangler deploy

text

#### 6. Visit

After successful deployment, visit your Worker URL:
https://flux-ai-pro.your-account.workers.dev

text

### 📜 Changelog

#### v9.6.0-gen-api (2025-12-17) 🎉 Latest
**🔥 Major Update - API Endpoint Migration**

**API Changes**
- ✅ **New Endpoint**: `https://image.pollinations.ai` → `https://gen.pollinations.ai`
- ✅ **URL Format**: `/prompt/{text}` → `/image/{text}`
- ✅ **Required Auth**: Added `Authorization: Bearer <token>` header
- ✅ **Error Handling**: Dedicated 401/403 authentication failure handling

**Authentication System**
- ✅ **API Key Required**: Read from `POLLINATIONS_API_KEY` environment variable
- ✅ **Bearer Token**: Standard OAuth 2.0 authentication
- ✅ **Status Display**: Real-time authentication status in UI
- ✅ **Warning Prompts**: Display warnings when API Key not set

**Breaking Changes** ⚠️
- ❌ **API Key Required**: Won't work properly without it
- ❌ **Old URL Deprecated**: Old `image.pollinations.ai` endpoint may no longer work
- ❌ **Anonymous Mode Limited**: Recommended to use API Key authentication

---

#### v9.5.2-bytes (2025-12-17)
- ✅ **Image bytes response**: Returns binary data instead of URLs
- ✅ Dual response mode: Single image / Batch generation
- ✅ HTTP Headers metadata transmission
- ✅ Base64 encoding support

---

#### v9.5.1-fixed (2025-12-17)
- ✅ Fixed CSP inline event errors
- ✅ Added Favicon (avoid 404)
- ✅ Fixed generation result display
- ✅ Improved history display

---

#### v9.5.0 (2025-12-16)
- 🎉 Initial release
- 🎨 4 official model support
- 🌐 Complete Web UI interface
- 🚀 Smart parameter optimization

---

### 📄 License

MIT License

### 🤝 Contributing

Issues and Pull Requests are welcome!

### 🔗 Links

- [Pollinations.ai](https://pollinations.ai/)
- [Pollinations API Docs](https://enter.pollinations.ai/api/docs)
- [Cloudflare Workers](https://workers.cloudflare.com/)
- [Project Homepage](https://github.com/kinai9661/Flux-AI-Pro)

---

<div align="center">

**Made with ❤️ by the community**

⭐ Star this repo if you find it helpful!

[Report Bug](https://github.com/kinai9661/Flux-AI-Pro/issues) · [Request Feature](https://github.com/kinai9661/Flux-AI-Pro/issues)

</div>
