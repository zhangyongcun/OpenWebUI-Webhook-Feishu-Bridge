# open-webui-webhook-feishu-bridge
## 解决什么问题？
[open-webui]([url](https://github.com/open-webui/open-webui)) 可设置新用户注册可以通过 webhook 发送通知到 Discord, Google Chat and Microsoft Teams，但不支持飞书消息。
该代码通过在 Cloudflare 创建 Workers 进行格式转换可以支持新注册用户时，发送消息到飞书。
## 操作步骤
1. 创建[飞书自定义机器人](https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot#461aa643)，复制地址并替换下面的代码中的地址；
2. [Cloudflare](dash.cloudflare.com) 中创建 worker 并粘贴以下代码；
3. 将 worker 的访问地址填入到[open-webui]([url](https://github.com/open-webui/open-webui)) ```管理员```-```通用```-```Webhook URL```中并保存；
4. 新建用户测试。

## 代码
> 注意修改 ```feishuUrl``` 变量为你自己的飞书自定义机器人地址。
```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  if (request.method !== "POST") {
    return new Response("Only POST method is allowed", { status: 405 });
  }

  try {
    const reqData = await request.json(); // 尝试解析接收到的 JSON 数据

    // 检查 user 字段是否存在且为有效的 JSON 字符串
    if (!reqData.user) {
      return new Response("Missing 'user' field in request data", { status: 400 });
    }

    // 尝试解析 user 字段中的字符串为 JSON 对象
    let userData;
    try {
      userData = JSON.parse(reqData.user);
    } catch (error) {
      return new Response("Invalid JSON format in 'user' field", { status: 400 });
    }

    // 确保 userData 包含必要的字段
    if (!userData.name || !userData.email) {
      return new Response("Missing required user information (name or email)", { status: 400 });
    }

    // 准备发送到飞书的数据
    const feishuData = JSON.stringify({
      "msg_type": "text",
      "content": {
        "text": `New user signed up: Name - ${userData.name}, Email - ${userData.email}`
      }
    });

    // 飞书机器人地址，修改为你自己的 webhook 地址
    const feishuUrl = 'https://open.feishu.cn/open-apis/bot/v2/hook/**************';

    const feishuResponse = await fetch(feishuUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: feishuData
    });

    if (feishuResponse.ok) {
      return new Response("Data forwarded to Feishu successfully", { status: 200 });
    } else {
      const errorMessage = await feishuResponse.text();
      return new Response(`Failed to forward data to Feishu: ${errorMessage}`, { status: 500 });
    }
  } catch (error) {
    return new Response(`Error processing the request: ${error.message}`, { status: 500 });
  }
}
```
