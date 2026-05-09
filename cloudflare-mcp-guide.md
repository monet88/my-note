# Advanced Stateless MCP Server on Cloudflare Workers (TypeScript)

Guide này hướng dẫn cách xây dựng và triển khai một MCP Server chuyên nghiệp trên Cloudflare Workers. Phương pháp này sử dụng **TypeScript**, **MCP SDK chính thức**, và hỗ trợ các tính năng nâng cao như **Xoay vòng API Key (Round-Robin)**.

## Tại sao chọn Cloudflare Workers?
- **Stateless HTTP:** Tương thích hoàn hảo với ChatGPT Custom Apps.
- **Tốc độ:** Cold start gần như bằng 0ms.
- **Chi phí:** Miễn phí lên tới 100,000 requests/ngày.
- **SDK chính thức:** Sử dụng `@modelcontextprotocol/sdk` thay vì viết code xử lý JSON-RPC thủ công.

## 1. Cấu trúc Project (Khuyến nghị)
```text
project-root/
├── src/                # Logic cốt lõi
│   ├── utils/          # Key Pool, Logger, Error Handler
│   └── tools/          # Định nghĩa các Tools
├── workers/            # Cloudflare Adapter
│   ├── src/
│   │   └── worker.ts   # Entry point của Worker
│   ├── wrangler.toml   # Cấu hình Cloudflare
│   └── tsconfig.json
└── package.json
```

## 2. Triển khai Stateless Adapter (`worker.ts`)

Bí quyết để chạy MCP SDK trên môi trường Stateless là truy cập trực tiếp vào internal registry của server.

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { initializeMcpServer, McpConfig } from '../../src/mcp-handler.js';
import { updateKeyPool } from '../../src/utils/build-key-pool.js';

export default {
  async fetch(request: Request, env: Env) {
    const url = new URL(request.url);

    // 1. Đồng bộ Key Pool từ Environment (Secrets)
    const envKeys = env.EXA_API_KEYS || env.EXA_API_KEY;
    if (envKeys) updateKeyPool(envKeys);

    // 2. Trích xuất API Key từ Request (nếu có)
    const requestKey = getRequestApiKey(request, url);

    // 3. Khởi tạo MCP Server với config
    const config: McpConfig = {
      exaApiKey: requestKey, // Nếu undefined, hệ thống sẽ dùng Key Pool tự động
      userProvidedApiKey: Boolean(requestKey),
    };
    
    const server = new McpServer({ name: "my-server", version: "1.0.0" });
    initializeMcpServer(server, config);

    // 4. Xử lý JSON-RPC (Tools List & Tools Call)
    // ... (Code xử lý router JSON-RPC)
  }
};
```

## 3. Cơ chế Xoay vòng API Key (Round-Robin)

Để tối ưu hóa rate limit, chúng ta sử dụng một `KeyPool` để quản lý danh sách API Key.

### Cấu hình Secrets:
Sử dụng `wrangler` để nạp danh sách key:
```bash
# Nạp chuỗi key phân cách bằng dấu phẩy
npx wrangler secret put EXA_API_KEYS
# Giá trị: key1,key2,key3
```

### Cách hoạt động:
- Nếu một key bị **429 (Rate Limited)**, hệ thống tự động đánh dấu cooldown và chuyển sang key tiếp theo.
- Nếu key bị **401/403 (Invalid)**, key đó sẽ bị loại bỏ khỏi pool.
- Việc xoay vòng giúp server hoạt động bền bỉ hơn khi có nhiều người dùng cùng lúc.

## 4. Deploy và Thiết lập

### Lệnh Deploy:
```bash
npm run cf:deploy
```

### URL Setup trên ChatGPT:
- **Endpoint:** `https://your-worker-name.<username>.workers.dev/mcp`
- **Method:** POST
- **Auth:** None (hoặc API Key qua Header `x-api-key`).

## 5. Mẹo nâng cao
1. **Batch Requests:** ChatGPT đôi khi gửi nhiều yêu cầu trong một mảng JSON. Hãy đảm bảo Worker của bạn có vòng lặp xử lý mảng này.
2. **CORS:** Luôn trả về đúng headers `Access-Control-Allow-Origin: *` để ChatGPT có thể gọi API từ trình duyệt.
3. **Logs:** Sử dụng `npx wrangler tail` để debug các lỗi phát sinh trong thời gian thực.
