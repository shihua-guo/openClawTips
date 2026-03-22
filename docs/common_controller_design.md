# CommonController 优化技术详细设计文档

## 1. 需求详情 (Requirement Details)

[CommonController](file:///C:/Users/shihu/Documents/workspace/RuoYi-Vue/ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java#28-163) 是系统中的通用文件处理控制器，负责文件的上传、下载及本地资源访问。为了提升系统的安全性、并发性能以及接口易用性，需要对现有实现进行优化。

### 1.1 主要需求
- **安全性增强**：防止目录穿越攻击（Path Traversal），限制非法文件类型上传，防止恶意大文件耗尽资源。
- **性能优化**：采用流式处理（Streaming）进行文件读写，减少大文件操作时的内存占用。
- **接口一致性**：统一异常返回格式，无论成功或失败，前端均能接收到标准化的 `AjaxResult`。
- **易用性提升**：优化多文件上传的返回结构，从逗号分隔的字符串改为标准的 JSON 数组，方便前端解析。

---

## 2. 功能逻辑 (Functional Logic)

### 2.1 文件下载逻辑 (Download Logic)
1. **合法性校验**：通过正则表达式严格校验文件名，禁止包含 `..` 或绝对路径字符。
2. **存在性检查**：在读取前确认文件在服务器物理路径上确实存在。
3. **流式输出**：使用 `BufferedInputStream` 读取文件并写入 `ServletOutputStream`，设置合适的缓冲区大小（如 8KB）。
4. **清理机制**：若请求参数 `delete` 为 `true`，在文件传输完成后异步或同步安全删除临时文件。

### 2.2 文件上传逻辑 (Upload Logic)
1. **类型过滤**：支持通过配置白名单（如 `png, jpg, pdf, docx`）过滤上传文件。
2. **大小限制**：在 Controller 层增加对 `MultipartFile` 大小的校验。
3. **命名混淆**：统一使用 UUID 或时间戳重命名文件，避免原始文件名冲突及潜在的安全风险。
4. **统一存储**：通过 `FileUploadUtils` 封装底层存储逻辑（本地/OSS/MinIO），Controller 仅负责业务分发。

---

## 3. 涉及接口 (Related Interfaces)

### 3.1 通用下载接口
- **接口地址**：`GET /common/download`
- **功能描述**：根据文件名下载系统临时生成的下载文件。
- **优化点**：
    - **安全**：增加 `FileUtils.isValidFilename` 深度检查。
    - **性能**：使用 `response.setBufferSize` 和流式写入，避免 `FileUtils.writeBytes` 的大对象拷贝。
    - **错误处理**：若文件不存在，返回 `application/json` 格式的错误摘要，而非破坏下载二进制流。

### 3.2 通用上传接口（单个）
- **接口地址**：`POST /common/upload`
- **参数说明**：`MultipartFile file`
- **优化点**：
    - **校验**：增加 MIME Type 校验（不仅是扩展名）。
    - **返回**：返回完整的资源元数据（原名、现名、URL、大小、MD5）。

### 3.3 通用上传接口（多个）
- **接口地址**：`POST /common/uploads`
- **参数说明**：`List<MultipartFile> files`
- **优化点**：
    - **并发**：若存储后端支持，可考虑并行执行上传任务（CompletableFuture）。
    - **返回结构**：**[重大改进]** 废弃 `urls` 逗号拼接字符串，返还 `List<Map<String, Object>>` 结构，包含每个文件的状态和 URL。

### 3.4 本地资源通用下载
- **接口地址**：`GET /common/download/resource`
- **功能描述**：用于下载数据库中存储的、位于 `profile` 路径下的静态资源。
- **优化点**：
    - **前缀自适应**：自动处理 `/profile` 前缀，防止路径拼接错误。
    - **缓存控制**：根据资源类型设置 HTTP 缓存头（Cache-Control），提升二次访问速度。
