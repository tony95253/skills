# Migration Guide - 從舊模式遷移到新架構

從傳統 Express 寫法遷移到本指南推薦的分層架構。

## 目錄

- [遷移策略](#遷移策略)
- [從路由內聯邏輯遷移](#從路由內聯邏輯遷移)
- [從直接 Prisma 遷移到 Repository](#從直接-prisma-遷移到-repository)
- [從 process.env 遷移到 unifiedConfig](#從-processenv-遷移到-unifiedconfig)
- [從 console.log 遷移到 Sentry](#從-consolelog-遷移到-sentry)
- [遷移檢查清單](#遷移檢查清單)

---

## 遷移策略

### 漸進式遷移原則

1. **不要一次全改**：按功能模組逐步遷移
2. **保持向後相容**：遷移期間新舊程式碼可共存
3. **測試優先**：每個遷移步驟都要有測試覆蓋
4. **優先處理高風險區域**：先遷移錯誤處理和驗證相關程式碼

### 建議遷移順序

```
1. 設定 Sentry (instrument.ts)
   ↓
2. 建立 unifiedConfig
   ↓
3. 建立 BaseController
   ↓
4. 遷移 Routes → Controller
   ↓
5. 抽取 Services
   ↓
6. 建立 Repositories
   ↓
7. 加入 Zod 驗證
```

---

## 從路由內聯邏輯遷移

### Before: 所有邏輯在路由中

```typescript
// ❌ 舊寫法：routes/users.ts
router.post('/users', async (req, res) => {
  try {
    // 驗證
    if (!req.body.email || !req.body.name) {
      return res.status(400).json({ error: 'Missing fields' })
    }

    // 業務邏輯
    const existingUser = await prisma.user.findUnique({
      where: { email: req.body.email }
    })
    if (existingUser) {
      return res.status(409).json({ error: 'Email exists' })
    }

    // 資料庫操作
    const user = await prisma.user.create({
      data: {
        email: req.body.email,
        name: req.body.name,
        createdAt: new Date()
      }
    })

    // 發送郵件
    await sendWelcomeEmail(user.email)

    res.status(201).json(user)
  } catch (error) {
    console.error('Failed to create user:', error)
    res.status(500).json({ error: 'Internal server error' })
  }
})
```

### After: 分層架構

**Step 1: 建立 Zod Schema**

```typescript
// validators/userValidators.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.string().email('Invalid email format'),
  name: z.string().min(1, 'Name is required').max(100)
})

export type CreateUserInput = z.infer<typeof createUserSchema>
```

**Step 2: 建立 Repository**

```typescript
// repositories/UserRepository.ts
import { PrismaClient, User } from '@prisma/client'

export class UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } })
  }

  async create(data: { email: string; name: string }): Promise<User> {
    return this.prisma.user.create({
      data: { ...data, createdAt: new Date() }
    })
  }
}
```

**Step 3: 建立 Service**

```typescript
// services/userService.ts
import * as Sentry from '@sentry/node'
import { UserRepository } from '../repositories/UserRepository'
import { CreateUserInput } from '../validators/userValidators'

export class UserService {
  constructor(
    private userRepo: UserRepository,
    private emailService: EmailService
  ) {}

  async createUser(input: CreateUserInput) {
    const existing = await this.userRepo.findByEmail(input.email)
    if (existing) {
      throw new ConflictError('Email already registered')
    }

    const user = await this.userRepo.create(input)

    // 非同步發送郵件，不阻塞回應
    this.emailService.sendWelcome(user.email).catch(error => {
      Sentry.captureException(error, {
        tags: { operation: 'welcome_email' },
        extra: { userId: user.id }
      })
    })

    return user
  }
}
```

**Step 4: 建立 Controller**

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express'
import { BaseController } from './BaseController'
import { UserService } from '../services/userService'
import { createUserSchema } from '../validators/userValidators'

export class UserController extends BaseController {
  constructor(private userService: UserService) {
    super()
  }

  async create(req: Request, res: Response): Promise<void> {
    try {
      const validated = createUserSchema.parse(req.body)
      const user = await this.userService.createUser(validated)
      this.handleSuccess(res, user, 201)
    } catch (error) {
      this.handleError(error, res, 'createUser')
    }
  }
}
```

**Step 5: 簡化路由**

```typescript
// routes/userRoutes.ts
import { Router } from 'express'
import { userController } from '../bootstrap'

const router = Router()

router.post('/users', (req, res) => userController.create(req, res))

export default router
```

---

## 從直接 Prisma 遷移到 Repository

### Before: 散落各處的 Prisma 調用

```typescript
// ❌ 舊寫法：直接在各處使用 prisma
// routes/posts.ts
const posts = await prisma.post.findMany({
  where: { authorId: userId, status: 'published' },
  orderBy: { createdAt: 'desc' },
  include: { author: true }
})

// services/analytics.ts
const count = await prisma.post.count({
  where: { authorId: userId, status: 'published' }
})

// controllers/dashboard.ts
const recentPosts = await prisma.post.findMany({
  where: { status: 'published' },
  take: 10,
  orderBy: { createdAt: 'desc' }
})
```

### After: Repository 統一管理

```typescript
// repositories/PostRepository.ts
import { PrismaClient, Post, Prisma } from '@prisma/client'

export class PostRepository {
  constructor(private prisma: PrismaClient) {}

  // 查詢條件可重用
  private publishedByAuthor(authorId: string): Prisma.PostWhereInput {
    return { authorId, status: 'published' }
  }

  async findPublishedByAuthor(authorId: string): Promise<Post[]> {
    return this.prisma.post.findMany({
      where: this.publishedByAuthor(authorId),
      orderBy: { createdAt: 'desc' },
      include: { author: true }
    })
  }

  async countPublishedByAuthor(authorId: string): Promise<number> {
    return this.prisma.post.count({
      where: this.publishedByAuthor(authorId)
    })
  }

  async findRecentPublished(limit: number = 10): Promise<Post[]> {
    return this.prisma.post.findMany({
      where: { status: 'published' },
      take: limit,
      orderBy: { createdAt: 'desc' }
    })
  }
}
```

### 遷移步驟

1. 識別所有 Prisma 調用位置
2. 按 Entity 分組（User, Post, Comment 等）
3. 建立對應的 Repository 類別
4. 逐一替換直接調用為 Repository 方法
5. 確保測試通過

---

## 從 process.env 遷移到 unifiedConfig

### Before: 散落的 process.env

```typescript
// ❌ 舊寫法：各處直接讀取環境變數
// database.ts
const client = new PrismaClient({
  datasources: {
    db: { url: process.env.DATABASE_URL }
  }
})

// email.ts
const apiKey = process.env.SENDGRID_API_KEY
const fromEmail = process.env.FROM_EMAIL || 'noreply@example.com'

// server.ts
const port = parseInt(process.env.PORT || '3000', 10)
const isProduction = process.env.NODE_ENV === 'production'
```

### After: 統一配置管理

**Step 1: 建立配置結構**

```typescript
// config/unifiedConfig.ts
import { z } from 'zod'

const configSchema = z.object({
  env: z.enum(['development', 'staging', 'production']),
  port: z.number().default(3000),

  database: z.object({
    url: z.string().url(),
    maxConnections: z.number().default(10)
  }),

  email: z.object({
    apiKey: z.string(),
    fromAddress: z.string().email().default('noreply@example.com')
  }),

  sentry: z.object({
    dsn: z.string().optional(),
    sampleRate: z.number().min(0).max(1).default(1.0)
  }),

  timeouts: z.object({
    default: z.number().default(30000),
    long: z.number().default(60000)
  })
})

function loadConfig() {
  const rawConfig = {
    env: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT || '3000', 10),

    database: {
      url: process.env.DATABASE_URL,
      maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS || '10', 10)
    },

    email: {
      apiKey: process.env.SENDGRID_API_KEY,
      fromAddress: process.env.FROM_EMAIL
    },

    sentry: {
      dsn: process.env.SENTRY_DSN,
      sampleRate: parseFloat(process.env.SENTRY_SAMPLE_RATE || '1.0')
    },

    timeouts: {
      default: parseInt(process.env.TIMEOUT_DEFAULT || '30000', 10),
      long: parseInt(process.env.TIMEOUT_LONG || '60000', 10)
    }
  }

  return configSchema.parse(rawConfig)
}

export const config = loadConfig()
export type Config = z.infer<typeof configSchema>
```

**Step 2: 替換所有 process.env**

```typescript
// ✅ 新寫法：使用 config
import { config } from './config/unifiedConfig'

// database.ts
const client = new PrismaClient({
  datasources: {
    db: { url: config.database.url }
  }
})

// email.ts
const apiKey = config.email.apiKey
const fromEmail = config.email.fromAddress

// server.ts
const port = config.port
const isProduction = config.env === 'production'
```

### 遷移步驟

1. 建立 `config/unifiedConfig.ts`
2. 搜尋所有 `process.env` 使用處
3. 逐一加入 config schema
4. 替換所有直接引用
5. 刪除不再使用的環境變數讀取

---

## 從 console.log 遷移到 Sentry

### Before: console.log 錯誤處理

```typescript
// ❌ 舊寫法
try {
  await riskyOperation()
} catch (error) {
  console.error('Operation failed:', error)
  // 錯誤被吞掉，無法追蹤
}
```

### After: Sentry 錯誤追蹤

**Step 1: 設定 instrument.ts（必須是第一個 import）**

```typescript
// instrument.ts - 必須在 app.ts 最頂部 import
import * as Sentry from '@sentry/node'
import { config } from './config/unifiedConfig'

Sentry.init({
  dsn: config.sentry.dsn,
  environment: config.env,
  sampleRate: config.sentry.sampleRate,
  integrations: [
    Sentry.httpIntegration(),
    Sentry.expressIntegration()
  ]
})

export { Sentry }
```

**Step 2: 在 app.ts 最頂部引入**

```typescript
// app.ts
import './instrument' // 必須是第一行
import express from 'express'
// ... 其他 imports
```

**Step 3: 替換錯誤處理**

```typescript
// ✅ 新寫法
import * as Sentry from '@sentry/node'

try {
  await riskyOperation()
} catch (error) {
  Sentry.captureException(error, {
    tags: {
      operation: 'riskyOperation',
      module: 'payment'
    },
    extra: {
      userId: currentUser.id,
      amount: paymentAmount
    }
  })
  throw error // 或適當處理
}
```

### BaseController 整合

```typescript
// controllers/BaseController.ts
import * as Sentry from '@sentry/node'

export abstract class BaseController {
  protected handleError(error: unknown, res: Response, operation: string): void {
    Sentry.captureException(error, {
      tags: { operation, controller: this.constructor.name }
    })

    if (error instanceof ValidationError) {
      res.status(400).json({ error: error.message })
    } else if (error instanceof NotFoundError) {
      res.status(404).json({ error: error.message })
    } else {
      res.status(500).json({ error: 'Internal server error' })
    }
  }

  protected handleSuccess(res: Response, data: unknown, status = 200): void {
    res.status(status).json(data)
  }
}
```

---

## 遷移檢查清單

### Phase 1: 基礎設施

- [ ] 設定 Sentry (`instrument.ts`)
- [ ] 建立 `unifiedConfig`
- [ ] 建立 `BaseController`
- [ ] 設定自訂錯誤類別

### Phase 2: 核心模組遷移

對每個功能模組重複以下步驟：

- [ ] 建立 Zod 驗證 Schema
- [ ] 建立 Repository（如需要）
- [ ] 建立 Service
- [ ] 建立 Controller（繼承 BaseController）
- [ ] 簡化 Routes
- [ ] 更新測試

### Phase 3: 清理

- [ ] 移除所有 `process.env` 直接引用
- [ ] 移除所有 `console.log/error`
- [ ] 移除路由中的業務邏輯
- [ ] 移除直接的 Prisma 調用（改用 Repository）

### Phase 4: 驗證

- [ ] 所有測試通過
- [ ] Sentry 收到測試錯誤
- [ ] API 回應格式一致
- [ ] 錯誤處理覆蓋所有端點

---

## 常見遷移問題

### Q: 簡單的 CRUD 也需要 Service 層嗎？

A: 對於純粹的資料存取，可以讓 Controller 直接調用 Repository。只有當有業務邏輯時才需要 Service。

### Q: 遷移期間如何處理新舊程式碼共存？

A:
1. 新功能使用新架構
2. 舊功能在修改時順便遷移
3. 使用 feature flags 控制切換

### Q: 如何處理循環依賴？

A:
1. 使用依賴注入容器
2. 將共用邏輯抽到獨立 utils
3. 重新審視模組邊界

---

**下一步**：閱讀 [complete-examples.md](complete-examples.md) 查看完整的遷移範例。
