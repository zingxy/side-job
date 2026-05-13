# 图片上传 — 前端接入指南

## 一句话概括

两步完成上传：**先调后端拿签名 → 再直传 OSS**，文件不经过后端服务器。

```
你的前端 ──① 请求签名──▶ 后端 :8080
你的前端 ◀── 签名+URL ── 后端
你的前端 ──② PUT 文件──▶ 阿里云 OSS（直连）
```

## 为什么要这样做

- 后端不中转文件流，节省带宽和内存
- 前端拿到 `publicUrl` 后可直接作为 `<img src>` 使用
- 前端传入唯一 `uniqueId`，文件名自动拼接，保证 URL 唯一

---

## 第一步：获取上传签名

向你的后端发一个 JSON POST：

```
POST /api/uploadSignUrl
Content-Type: application/json

{
  "fileName": "avatar.png",
  "contentType": "image/png",
  "uniqueId": 1712345678000
}
```

`uniqueId` 由前端生成（如 `Date.now()`），用于保证文件名唯一。后端会将其拼接到文件名中并原样返回，前端可通过它做请求-响应匹配。

后端返回：

```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "fileName": "avatar.png",
    "uniqueId": 1712345678000,
    "uploadUrl": "https://bucket.oss-cn-hangzhou.aliyuncs.com/images/...?Expires=xxx&Signature=xxx",
    "objectKey": "images/2026/05/10/avatar_1712345678000.png",
    "publicUrl": "https://bucket.oss-cn-hangzhou.aliyuncs.com/images/2026/05/10/avatar_1712345678000.png"
  }
}
```

你只需要关注三个字段：
- **uploadUrl** — 第二步 PUT 的地址（带签名，15 分钟有效）
- **publicUrl** — 上传完成后图片的永久公开地址
- **objectKey** — 文件在 OSS 的路径（**暂无用处，但保留，避免后面需要生成限时的publicUrl图片**）

---

## 第二步：直传文件到 OSS

用第一步拿到的 `uploadUrl`，直接 PUT 文件二进制：

```
PUT <uploadUrl>
Content-Type: image/png          ← 必须和第一步的 contentType 一致
Cache-Control: public, max-age=31536000
Content-Disposition: inline

[文件二进制数据]
```

返回 HTTP 200 即表示上传成功。

> 重点：`Content-Type` 必须和获取签名时传的一致，否则 OSS 会报签名不匹配。

---

## 完整前端代码示例

参考项目中的 `webSource/html/upload.html`，核心逻辑如下：

```javascript
// 追踪所有待处理的请求，支持并发上传
const pendingRequests = new Map();

async function uploadImage(file) {
    // 1. 拿签名（uniqueId 由前端生成，保证文件名唯一）
    const uniqueId = Date.now();
    pendingRequests.set(uniqueId, { fileName: file.name });

    const signRes = await fetch('/api/uploadSignUrl', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            fileName: file.name,
            contentType: file.type,
            uniqueId: uniqueId
        })
    });
    const signData = await signRes.json();

    // 校验响应匹配：确认返回的是本次请求的结果
    if (!pendingRequests.has(uniqueId)) {
        throw new Error('该上传已取消');
    }
    if (signData.data.uniqueId !== uniqueId || signData.data.fileName !== file.name) {
        pendingRequests.delete(uniqueId);
        throw new Error('响应不匹配，可能收到其他请求的结果');
    }
    pendingRequests.delete(uniqueId);

    if (signData.code !== 200) {
        throw new Error('获取签名失败: ' + signData.msg);
    }

    // 2. 直传 OSS
    const uploadRes = await fetch(signData.data.uploadUrl, {
        method: 'PUT',
        headers: {
            'Content-Type': file.type,
            'Cache-Control': 'public, max-age=31536000',
            'Content-Disposition': 'inline'
        },
        body: file  // HTML File 对象
    });

    if (!uploadRes.ok) {
        throw new Error('上传失败, HTTP ' + uploadRes.status);
    }

    // 3. 拿到永久 URL
    return signData.data.publicUrl;
}
```

---

## 删除图片

**当收到oss的回调结果，图片上传成功后，调用删除图片接口及时清理不使用的oss图片资源，避免oss容量不足**

```
POST /api/deleteImage
Content-Type: application/json

{
  "url": "https://bucket.oss-cn-hangzhou.aliyuncs.com/images/2026/05/10/avatar_1712345678.png"
}
```

成功返回 `{ "code": 200, "msg": "deleted", "data": { "objectKey": "..." } }`

文件不存在返回 `{ "code": 404, "msg": "file not found: ..." }`

> `url` 字段传完整 `publicUrl`，后端会自动解析。

---

## 常见错误

| 现象 | 原因 |
|------|------|
| OSS 返回 `SignatureDoesNotMatch` | PUT 请求的 `Content-Type` 和签名时不一致 |
| `uploadUrl` 返回 403 | 签名已过期（超过 15 分钟），需要重新获取 |
| 上传成功但 publicUrl 访问不了 | Bucket 没设为公共读，联系后端确认 |
| 浏览器 CORS 报错 | OSS 需要配置 CORS 规则，允许前端的域名做 PUT 请求 |
