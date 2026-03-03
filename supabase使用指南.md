Supabase 是一个开源的 Firebase 替代方案，基于 **PostgreSQL** 构建。对于个人调试和测试来说，它非常友好，因为它提供了免费的层级（Free Tier），并且无需配置复杂的服务器即可快速拥有一个功能完整的后端。

以下是针对个人调试测试的 Supabase 核心功能简介和使用指南：

### 1. 核心概念
*   **本质**：它是一个托管的 PostgreSQL 数据库，外加一套自动生成的 API 和工具链。
*   **优势**：你不需要写后端代码（如 Node.js/Python 服务）就能直接在前端代码中安全地读写数据库。

### 2. 主要功能模块（调试测试最常用）

#### A. 表格编辑器 (Table Editor)
*   **功能**：类似 Excel 或 Airtable 的界面，可以直接在网页上创建表、添加列、插入/修改数据。
*   **调试用途**：
    *   快速创建测试数据表（如 `users`, `todos`）。
    *   手动插入几条测试数据，验证前端读取是否正确。
    *   直接查看数据库中的原始数据，排查逻辑错误。

#### B. 自动生成的 API (Auto APIs)
*   **功能**：当你创建好表格后，Supabase 会自动为每张表生成 RESTful API 和类型安全的客户端库（Client Libraries）。
*   **调试用途**：
    *   **无需写 SQL**：直接使用 JavaScript/TypeScript/Python 等客户端 SDK 进行增删改查。
    *   **示例代码**：
      ```javascript
      // 查询数据
      const { data, error } = await supabase
        .from('todos')
        .select('*')
        .eq('is_complete', false);
      
      // 插入数据
      const { data, error } = await supabase
        .from('todos')
        .insert({ title: '测试任务', is_complete: false });
      ```

#### C. 身份验证 (Authentication / Auth)
*   **功能**：内置了完整的用户系统，支持邮箱/密码登录、魔法链接、以及第三方登录（Google, GitHub 等）。
*   **调试用途**：
    *   快速测试“用户登录”、“注册”、“重置密码”流程。
    *   利用 `Row Level Security (RLS)` 测试数据权限（例如：只允许用户看到自己的数据）。
    *   在 Dashboard 中手动创建测试用户，或查看当前在线用户。

#### D. 实时订阅 (Realtime)
*   **功能**：基于 PostgreSQL 的复制功能，允许前端监听数据库的变化。
*   **调试用途**：
    *   测试聊天室、即时通知或多用户协作功能。
    *   当数据库某张表发生变化时，前端页面自动更新，无需刷新。

#### E. SQL 编辑器 (SQL Editor)
*   **功能**：内置的代码编辑器，可以直接运行原生 SQL 语句。
*   **调试用途**：
    *   执行复杂的查询或批量数据处理。
    *   编写存储过程或触发器（Trigger）进行高级测试。
    *   一键保存常用的查询脚本。

#### F. 存储 (Storage)
*   **功能**：对象存储服务（类似 AWS S3），用于存图片、视频、文档。
*   **调试用途**：
    *   测试文件上传、下载功能。
    *   设置公共桶（Public Bucket）或私有桶的访问权限。

---

### 3. 快速开始流程（5分钟上手）

1.  **注册账号**：访问 [supabase.com](https://supabase.com) 并登录。
2.  **新建项目**：
    *   点击 "New Project"。
    *   填写项目名称、数据库密码（重要，请记好）、选择区域（选离你最近的，如 Tokyo 或 Singapore 以减小延迟）。
    *   等待几分钟初始化完成。
3.  **获取密钥**：
    *   进入项目后，点击左侧菜单 **Settings (齿轮图标)** -> **API**。
    *   你会看到 `Project URL` 和 `anon public` key（用于前端公开访问）。
    *   *注意：还有一个 `service_role` key，权限极高，仅限服务器端使用，调试时小心不要泄露给前端。*
4.  **连接客户端**：
    *   在你的本地项目中安装 SDK：
      ```bash
      npm install @supabase/supabase-js
      ```
    *   初始化连接：
      ```javascript
      import { createClient } from '@supabase/supabase-js'

      const supabaseUrl = '你的项目URL'
      const supabaseKey = '你的anon key'
      const supabase = createClient(supabaseUrl, supabaseKey)
      ```
5.  **开始测试**：
    *   去 **Table Editor** 建个表。
    *   去 **Auth** 开个注册用户。
    *   在本地代码里试着读写一下数据。

### 4. 个人调试特别提示

*   **免费额度足够用**：免费版提供 500MB 数据库空间、1GB 文件存储和充足的带宽，完全够个人开发和测试小项目。
*   **暂停与恢复**：如果项目长期不用，可以在后台暂停项目以释放资源（虽然免费版通常不需要，但了解此功能很好）。
*   **RLS (行级安全)**：这是 Supabase 的核心安全机制。**务必开启 RLS** 并设置策略，否则你的数据可能对全网公开。在测试初期，可以先关闭 RLS 方便调试，但在模拟真实环境前一定要开启并配置规则。
*   **本地开发**：如果你需要离线调试或更深层的控制，Supabase 提供了 CLI 工具 (`supabase start`)，可以在本地 Docker 中运行一个完整的 Supabase 实例。

总结来说，对于个人测试，你只需要关注：**建表 -> 设权限 (RLS) -> 用 SDK 读写**。剩下的复杂运维工作 Supabase 都帮你自动化了。
