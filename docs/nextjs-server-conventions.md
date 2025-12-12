# Next.js 服务端代码架构与命名规范 (Market Standards)

本文档总结了目前市面上（2024-2025）主流的 Next.js (App Router) 服务端代码实现包命名和路径规则。旨在统一开发规范，提升代码的可维护性和扩展性。

## 1. 核心目录结构 (Directory Structure)

在 `src/` 目录下，服务端代码通常按照 **功能模块 (Feature)** 或 **技术分层 (Layer)** 进行组织。目前最流行的是 **混合结构**。

```
src/
├── app/                  # 路由与页面 (Presentation Layer)
│   ├── api/              # HTTP API Routes (用于对外暴露接口)
│   └── (routes)/         # 页面路由
├── actions/              # Server Actions (服务端入口，替代 API Routes 处理表单/变更)
├── services/             # 业务逻辑层 (Business Logic / Service Layer)
├── data/ (或 db/queries) # 数据访问层 (Data Access Layer / DAL)
├── lib/                  # 第三方库封装 (Third-party integrations: Prisma, Stripe, Auth)
├── utils/                # 通用工具函数 (Helpers: date, math, string)
└── types/ (或 dto/)      # 类型定义与数据传输对象
```

---

## 2. 详细分层与命名规则

### 2.1 Server Actions (服务端操作入口)
Server Actions 是 Next.js 推荐的处理数据变更（Mutations）的方式。

*   **路径:** `src/actions/`
*   **命名规则:** `[feature]-actions.ts` 或 `[domain].ts` (文件名使用 kebab-case)
*   **函数命名:** `verbNoun` (camelCase)
*   **示例:**
    *   `src/actions/auth-actions.ts` -> `login`, `register`, `logout`
    *   `src/actions/user-actions.ts` -> `updateUserProfile`, `deleteAccount`

### 2.2 Service Layer (业务逻辑层)
当业务逻辑复杂，或者需要在 Server Actions 和 API Routes 之间复用逻辑时，引入 Service 层。

*   **路径:** `src/services/`
*   **命名规则:** `[feature].service.ts` 或简单使用 `[feature].ts`
*   **实现方式:**
    *   **函数式 (推荐):** 导出一组独立的函数。
    *   **类 (Class):** 适用于需要依赖注入的场景。
*   **示例:**
    *   `src/services/payment.service.ts`
    *   `src/services/order.ts`

### 2.3 Data Access Layer (数据访问层)
专注于数据库查询，不包含复杂业务逻辑。通常直接调用 ORM (Prisma, Drizzle) 或 SQL。

*   **路径:** `src/data/` 或 `src/db/queries/`
*   **命名规则:** `[feature].ts` 或 `[entity].ts`
*   **函数命名:** `get[Entity]`, `find[Entity]By[Field]`
*   **示例:**
    *   `src/data/user.ts` -> `getUserByEmail`, `getUserById`
    *   `src/data/analytics.ts` -> `getDailyActiveUsers`

### 2.4 Lib (第三方库配置与封装)
存放单例 (Singleton) 客户端或特定库的配置。

*   **路径:** `src/lib/`
*   **示例:**
    *   `src/lib/prisma.ts` (Prisma Client 实例)
    *   `src/lib/redis.ts`
    *   `src/lib/stripe.ts`

---

## 3. 文件命名规范 (File Naming Conventions)

| 类型 | 规则 | 示例 | 备注 |
| :--- | :--- | :--- | :--- |
| **文件/文件夹** | **kebab-case** | `user-profile.ts`, `api-utils.ts` | 确保跨操作系统兼容性，URL 友好 |
| **React 组件** | **PascalCase** | `UserProfile.tsx`, `Button.tsx` | 组件文件通常首字母大写 (也有团队用 kebab-case) |
| **Server Actions** | **kebab-case** | `create-post.ts` | 内部导出函数用 camelCase |
| **工具函数** | **kebab-case** | `date-formatter.ts` | 内部导出函数用 camelCase |
| **类型定义** | **kebab-case** | `user-types.ts` | 内部 Interface/Type 用 PascalCase |

---

## 4. 代码实现示例

### 场景：用户修改个人资料

**1. Data Layer (`src/data/user.ts`)**
```typescript
import { db } from "@/lib/db";

export async function getUserById(id: string) {
  return await db.user.findUnique({ where: { id } });
}

export async function updateUserInDb(id: string, data: any) {
  return await db.user.update({ where: { id }, data });
}
```

**2. Service Layer (`src/services/user-service.ts`)**
*可选层，如果逻辑简单可跳过直接写在 Action 中*
```typescript
import { updateUserInDb } from "@/data/user";
import { revalidatePath } from "next/cache";

export async function updateProfileLogic(userId: string, newData: any) {
  // 业务校验
  if (!newData.email.includes("@")) throw new Error("Invalid email");

  const updatedUser = await updateUserInDb(userId, newData);

  // 触发缓存更新等副作用
  revalidatePath("/dashboard/profile");
  return updatedUser;
}
```

**3. Server Action (`src/actions/user-actions.ts`)**
```typescript
"use server";

import { updateProfileLogic } from "@/services/user-service";
import { auth } from "@/auth"; // 假设的 auth 库

export async function updateProfile(formData: FormData) {
  const session = await auth();
  if (!session) return { error: "Unauthorized" };

  const rawData = Object.fromEntries(formData.entries());

  try {
    await updateProfileLogic(session.user.id, rawData);
    return { success: true };
  } catch (err) {
    return { error: "Update failed" };
  }
}
```
