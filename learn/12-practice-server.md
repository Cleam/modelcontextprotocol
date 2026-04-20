# 🛠️ 12 — 实战：构建 MCP Server

## 目标

本章带你从零构建一个完整的 MCP Server——一个**天气查询服务**，暴露工具、资源和提示词模板三种能力。通过这个实战项目，你将理解：

- MCP Server 的项目结构
- 如何定义和实现 Tools
- 如何提供 Resources
- 如何创建 Prompts
- 如何与 Claude Desktop 集成测试

---

## 📋 准备工作

### 技术栈

我们提供 **Python** 和 **TypeScript** 两个版本的实现。

**Python 版本需要**：

- Python 3.10+
- `mcp` SDK（`pip install mcp`）
- `httpx`（HTTP 客户端，`pip install httpx`）

**TypeScript 版本需要**：

- Node.js 18+
- `@modelcontextprotocol/sdk`

---

## 🐍 Python 版本

### 项目结构

```
weather-server/
├── pyproject.toml
├── src/
│   └── weather_server/
│       ├── __init__.py
│       └── server.py
```

### 第 1 步：创建项目

```bash
# 创建项目目录
mkdir weather-server && cd weather-server

# 初始化 Python 项目（推荐使用 uv）
uv init --name weather-server
uv add mcp httpx
```

或者使用 pip：

```bash
mkdir weather-server && cd weather-server
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install mcp httpx
```

### 第 2 步：实现 Server

```python
"""
weather_server/server.py
一个完整的天气查询 MCP Server，演示 Tools / Resources / Prompts 三种能力。
"""

import json
import httpx
from mcp.server.fastmcp import FastMCP

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 创建 FastMCP 实例
# FastMCP 是 Python SDK 提供的高级 API，
# 通过装饰器自动从函数签名和 docstring 生成协议定义。
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
mcp = FastMCP("weather-server")

# 国家气象服务 API 基础 URL
NWS_BASE = "https://api.weather.gov"

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 辅助函数
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

async def make_nws_request(url: str) -> dict | None:
    """向国家气象服务 API 发送请求。

    使用 httpx 异步客户端发送 GET 请求。
    NWS API 要求设置 User-Agent 头。
    """
    headers = {
        "User-Agent": "MCP-Weather-Server/1.0",
        "Accept": "application/geo+json",
    }
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=10.0)
            response.raise_for_status()
            return response.json()
        except (httpx.HTTPError, json.JSONDecodeError):
            return None


def format_alert(alert: dict) -> str:
    """将天气预警数据格式化为可读字符串。"""
    props = alert.get("properties", {})
    return f"""
⚠️ {props.get('event', '未知事件')}
区域: {props.get('areaDesc', '未知')}
严重程度: {props.get('severity', '未知')}
状态: {props.get('status', '未知')}
标题: {props.get('headline', '无')}
描述: {props.get('description', '无描述')}
""".strip()


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 定义 Tools（工具）
# 使用 @mcp.tool() 装饰器，FastMCP 会自动：
# 1. 从函数名生成工具名称
# 2. 从参数类型注解生成 inputSchema
# 3. 从 docstring 生成 description
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@mcp.tool()
async def get_alerts(state: str) -> str:
    """获取美国指定州的天气预警信息。

    Args:
        state: 美国州的两字母缩写，如 CA（加利福尼亚）、NY（纽约）
    """
    url = f"{NWS_BASE}/alerts/active?area={state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "无法获取预警信息，或该州当前没有活跃预警。"

    alerts = data["features"]
    if not alerts:
        return f"{state} 当前没有活跃的天气预警。"

    return "\n---\n".join(format_alert(alert) for alert in alerts[:5])


@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """获取指定经纬度位置的天气预报。

    使用两步 API 调用：先获取网格点信息，再获取详细预报。

    Args:
        latitude: 纬度，如 39.7456（丹佛）
        longitude: 经度，如 -104.9994（丹佛）
    """
    # 第 1 步：获取网格点信息
    points_url = f"{NWS_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return f"无法获取坐标 ({latitude}, {longitude}) 的网格点信息。"

    # 第 2 步：获取预报
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "无法获取天气预报数据。"

    # 格式化输出
    periods = forecast_data["properties"]["periods"][:5]
    forecasts = []
    for period in periods:
        forecasts.append(
            f"📅 {period['name']}:\n"
            f"   🌡️ 温度: {period['temperature']}°{period['temperatureUnit']}\n"
            f"   💨 风: {period['windSpeed']} {period['windDirection']}\n"
            f"   📝 {period['shortForecast']}"
        )
    return "\n\n".join(forecasts)


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 定义 Resources（资源）
# 使用 @mcp.resource() 装饰器定义可读取的数据源
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@mcp.resource("weather://supported-states")
async def get_supported_states() -> str:
    """返回支持查询的美国州列表。"""
    states = {
        "AL": "Alabama", "AK": "Alaska", "AZ": "Arizona",
        "CA": "California", "CO": "Colorado", "FL": "Florida",
        "NY": "New York", "TX": "Texas", "WA": "Washington",
    }
    return json.dumps(states, indent=2)


@mcp.resource("weather://api-info")
async def get_api_info() -> str:
    """返回天气 API 的使用说明。"""
    return """
天气查询 MCP Server API 说明
============================

本服务提供以下工具：

1. get_alerts(state)
   - 获取指定州的天气预警
   - 参数：州的两字母缩写（如 CA、NY）

2. get_forecast(latitude, longitude)
   - 获取指定坐标的天气预报
   - 参数：纬度和经度

数据来源：美国国家气象服务 (NWS) API
API 文档：https://www.weather.gov/documentation/services-web-api
"""


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 定义 Prompts（提示词模板）
# 使用 @mcp.prompt() 装饰器定义预设工作流
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@mcp.prompt()
async def weather_briefing(state: str) -> str:
    """生成一份天气简报的提示词模板。

    Args:
        state: 要查询的州（两字母缩写）
    """
    return f"""请为 {state} 州生成一份完整的天气简报，包含以下内容：

1. 首先使用 get_alerts 工具查询 {state} 的天气预警
2. 对于每个预警区域，使用 get_forecast 获取详细预报
3. 整理以下信息：
   - 当前活跃的天气预警（按严重程度排序）
   - 未来 24 小时天气概况
   - 出行建议

请以清晰、专业的格式输出。"""


@mcp.prompt()
async def travel_weather(destination: str, days: int = 3) -> str:
    """为旅行目的地生成天气分析的提示词模板。

    Args:
        destination: 目的地名称
        days: 旅行天数（默认 3 天）
    """
    return f"""我计划去 {destination} 旅行 {days} 天，请帮我分析天气情况：

1. 查询该地区的天气预报
2. 分析以下方面：
   - 是否有恶劣天气预警？
   - 适合的穿着建议
   - 是否需要携带雨具？
   - 对户外活动的影响

请给出具体、实用的建议。"""


# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 启动 Server
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

if __name__ == "__main__":
    # transport="stdio" 表示使用标准输入/输出传输
    mcp.run(transport="stdio")
```

### 第 3 步：测试运行

```bash
# 直接运行（检查是否有语法错误）
python src/weather_server/server.py

# 使用 MCP Inspector 进行交互式测试
npx @modelcontextprotocol/inspector python src/weather_server/server.py
```

MCP Inspector 提供了可视化的测试界面，可以：

- 查看 Server 暴露的所有 Tools / Resources / Prompts
- 手动调用工具并查看结果
- 检查协议消息的来回传递

### 第 4 步：接入 Claude Desktop

编辑 Claude Desktop 的配置文件：

**macOS**：`~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows**：`%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/absolute/path/to/src/weather_server/server.py"]
    }
  }
}
```

重启 Claude Desktop，你应该能在输入框旁看到 MCP Server 的指示器。

---

## 📘 TypeScript 版本

### 项目结构

```
weather-server-ts/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

### 实现

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// 创建 Server 实例
const server = new McpServer({
  name: "weather-server",
  version: "1.0.0",
});

const NWS_BASE = "https://api.weather.gov";

// 辅助函数
async function makeNwsRequest(url: string): Promise<any | null> {
  try {
    const response = await fetch(url, {
      headers: {
        "User-Agent": "MCP-Weather-Server/1.0",
        Accept: "application/geo+json",
      },
    });
    if (!response.ok) return null;
    return await response.json();
  } catch {
    return null;
  }
}

// ━━━━━━━━━━━━ 定义 Tools ━━━━━━━━━━━━

server.tool(
  "get_alerts",
  "获取美国指定州的天气预警",
  {
    state: z.string().length(2).describe("州的两字母缩写，如 CA、NY"),
  },
  async ({ state }) => {
    const data = await makeNwsRequest(
      `${NWS_BASE}/alerts/active?area=${state}`,
    );
    if (!data?.features?.length) {
      return {
        content: [{ type: "text", text: `${state} 当前没有活跃的天气预警。` }],
      };
    }
    const alerts = data.features
      .slice(0, 5)
      .map((a: any) => {
        const p = a.properties;
        return `⚠️ ${p.event}\n区域: ${p.areaDesc}\n严重程度: ${p.severity}`;
      })
      .join("\n---\n");

    return { content: [{ type: "text", text: alerts }] };
  },
);

server.tool(
  "get_forecast",
  "获取指定经纬度位置的天气预报",
  {
    latitude: z.number().describe("纬度"),
    longitude: z.number().describe("经度"),
  },
  async ({ latitude, longitude }) => {
    const pointsData = await makeNwsRequest(
      `${NWS_BASE}/points/${latitude},${longitude}`,
    );
    if (!pointsData) {
      return {
        content: [{ type: "text", text: "无法获取网格点信息。" }],
        isError: true,
      };
    }
    const forecastData = await makeNwsRequest(pointsData.properties.forecast);
    if (!forecastData) {
      return {
        content: [{ type: "text", text: "无法获取预报数据。" }],
        isError: true,
      };
    }
    const forecasts = forecastData.properties.periods
      .slice(0, 5)
      .map(
        (p: any) =>
          `📅 ${p.name}: ${p.temperature}°${p.temperatureUnit}, ${p.shortForecast}`,
      )
      .join("\n");

    return { content: [{ type: "text", text: forecasts }] };
  },
);

// ━━━━━━━━━━━━ 定义 Resources ━━━━━━━━━━━━

server.resource(
  "supported-states",
  "weather://supported-states",
  async (uri) => ({
    contents: [
      {
        uri: uri.href,
        mimeType: "application/json",
        text: JSON.stringify({
          CA: "California",
          NY: "New York",
          TX: "Texas",
        }),
      },
    ],
  }),
);

// ━━━━━━━━━━━━ 定义 Prompts ━━━━━━━━━━━━

server.prompt(
  "weather_briefing",
  "生成指定州的天气简报",
  [{ name: "state", description: "州的两字母缩写", required: true }],
  async ({ state }) => ({
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: `请为 ${state} 州生成天气简报，使用 get_alerts 和 get_forecast 工具。`,
        },
      },
    ],
  }),
);

// ━━━━━━━━━━━━ 启动 Server ━━━━━━━━━━━━

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## ⚠️ 常见问题

### 1. stdout 被污染

```python
# ❌ 这会让 Client 解析失败
print("Server started!")

# ✅ 日志输出到 stderr
import sys
print("Server started!", file=sys.stderr)

# ✅ 或使用 logging 模块
import logging
logging.basicConfig(stream=sys.stderr)
logger = logging.getLogger(__name__)
logger.info("Server started!")
```

### 2. 工具超时

如果你的工具需要较长时间执行，发送进度通知让 Client 知道你还在干活：

```python
@mcp.tool()
async def long_running_task(query: str, ctx: Context) -> str:
    """一个耗时的任务。"""
    await ctx.report_progress(0, 100, "开始处理...")

    # 模拟耗时操作
    result = await slow_operation_step1()
    await ctx.report_progress(50, 100, "处理中...")

    result = await slow_operation_step2(result)
    await ctx.report_progress(100, 100, "完成")

    return result
```

### 3. 调试技巧

```bash
# 使用 MCP Inspector（推荐）
npx @modelcontextprotocol/inspector python server.py

# 查看 Claude Desktop 的日志
# macOS:
tail -f ~/Library/Logs/Claude/mcp*.log

# 手动测试 stdin/stdout（高级）
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | python server.py
```

---

## 📌 小结

| 步骤           | 要点                                        |
| -------------- | ------------------------------------------- |
| 项目初始化     | 安装 MCP SDK + 依赖                         |
| 定义 Tools     | `@mcp.tool()` 装饰器 + 类型注解 + docstring |
| 定义 Resources | `@mcp.resource(uri)` 装饰器                 |
| 定义 Prompts   | `@mcp.prompt()` 装饰器                      |
| 测试           | MCP Inspector 可视化调试                    |
| 集成           | Claude Desktop 配置文件                     |
| 日志           | 只能输出到 stderr，不能输出到 stdout        |

---

> 📖 **下一章**：[实战：构建 MCP Client](./13-practice-client.md) — 实现一个能连接 MCP Server 的客户端。
