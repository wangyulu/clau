---
name: OpenAPI Documentation Generator
description: 一个根据 MySQL 表结构自动生成 OpenAPI 文档的工具，当用户需要生成 OpenAPI 文档时调用。
allowed-tools: Read, Grep, Glob
disable-model-invocation: false
---

# OpenAPI Skill 生成规范

## 1. 基础映射规则

### 1.1 数据类型映射

- varchar, char, text -> string
- bigint, int, tinyint, smallint -> integer (注意：bigint 需标注 format: int64)
- decimal, float, double -> number
- datetime, timestamp -> string (format: date-time)
- date → string (format: date)
- **特殊规则**：字段名以 _at 结尾者，无论数据库类型为何，统一映射为 string (format: date-time)
- json 字段映射：
  - 若 COMMENT 中包含 [] 或明确提及数组/列表，映射为 array。
  - 否则映射为 object。

### 1.2 枚举识别

- 自动解析字段 COMMENT 中的枚举定义。
- 格式通常为：字段说明：key-value, key-value... 或 字段说明：value-说明, value-说明...。
- 提取所有 key 值生成 OpenAPI enum 列表。

### 1.3 注释规范

- 所有 Schema 字段的 description 必须严格等于数据库 Schema 中的 COMMENT 内容，不做任何修改或省略。

### 1.4 全局配置

- **servers**：默认生成一个 url: / 的服务地址。
- **tags**：根据表名或业务模块名称生成 Tag，用于接口分组。

### 1.5 必填字段规则

- 数据库 NOT NULL 且无 DEFAULT 值 → required: true
- 创建接口：排除自增主键、时间戳字段后，其余 NOT NULL 字段为必填
- 更新接口：仅 id 为必填
- 有 DEFAULT 值的字段在 schema 中添加 default 属性

### 1.6 外键关联规则

- 外键字段命名为 {table}_id 时，自动识别关联关系
- 在详情接口的响应中，可选择性展开关联对象（如 user_id → user: {...}）
- 关联字段的 description 添加引用说明：如 "用户ID (关联 users 表)"

## 2.接口生成规则

- **2.1 路由设计**
  - **列表查询**：GET /{resource_name}
  - **详情查询**：GET /{resource_name}/detail
  - **创建**：POST /{resource_name}/create
  - **更新**：POST /{resource_name}/update
  - **删除**：POST /{resource_name}/delete
  - **批量删除**：POST /{resource_name}/batch-delete

- **2.2 请求参数规则**
  - **GET 请求**：参数置于 query 中。
    - **列表查询参数规则**
        - **基础参数**：
          - page (integer, default: 1)
          - page_size (integer, default: 20, maximum: 100)
          - sort_by (string): 排序字段
          - sort_order (string, enum: [asc, desc])

        - **筛选参数生成规则**：
          - 包含：所有非主键、非时间戳、非大文本(text/longtext)、非JSON字段
          - 字符串字段：支持模糊查询，参数名添加 _like 后缀（仅表结构Comment中明确说明支持模糊查询时才添加 _like 后缀）
          - 数值/日期字段：支持范围查询，生成 {field}_min 和 {field}_max（created_at字段必须添加 _min 和 _max 后缀)
          - 枚举字段：直接使用 enum 约束
        
        - **排除字段明确定义**：
          - 主键：id
          - 自动时间戳：updated_at, deleted_at
          - 大文本：类型为 text, mediumtext, longtext
          - JSON：类型为 json
  
    - **POST 请求**：参数置于 body (application/json) 中。
      - 创建：排除 id、created_at、updated_at、deleted_at。
      - 更新：包含 id 及需更新字段（扁平化结构）。
      - 删除：仅包含 id。
      - 批量删除：仅包含 ids。

- **2.3 响应结构标准化**

  - **标准外层包装**：
    - code (integer): 状态码。
    - msg (string): 提示信息。
    - data (object/null): 业务数据。
  - **列表数据结构** (data 内部)：
    - total (integer): 总记录数。
    - items (array): 数据列表。
    - **操作类响应**：data 为 null。

## 3. 组件复用规则

- 定义 {Model}Input：用于创建请求。
- 定义 {Model}UpdateRequest：用于更新请求。
- 定义 {Model}DeleteRequest：用于删除请求。
- 定义 {Model}：基础数据模型。
- 定义 ApiResponseList、ApiResponseDetail、ApiResponseAction：通用响应包装器。

## 4. 示例生成

- 所有接口必须提供完整的 example。
- 示例数据必须符合数据模型定义（类型、枚举值等）。
- **特殊**：
    - 列表接口：需包含除（大文本/json字段外）所有模型字段。
    - 创建接口：需包含除（id、created_at、updated_at、deleted_at）外所有模型字段。
    - 更新接口：需包含除（created_at、updated_at、deleted_at）外所有模型字段。

- **示例值生成规则**：
  - string: 根据字段名生成语义化值
      - name → "张三"
      - email → "user@example.com"
      - phone → "13800138000"
  - integer: 使用合理范围值 (id → 1, age → 25)
  - datetime: 使用当前时间或固定示例 "2024-01-01 00:00:00"
  - enum: 使用第一个枚举值
  - boolean: 根据语义选择 (is_active → true)
