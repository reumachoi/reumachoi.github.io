---
title: SSE 방식 MCP 서버 TypeScript, Python으로 만들기 (w.K8s)
author: rumi
date: 2025-05-06
categories: [AI, MCP]
tags: [claude, genai, mcp-server]
---

# 개요
MCP(Model Context Protocol) 서버를 구축하며 익힌 개념과 방법을 정리했습니다.  

## MCP란
[MCP 공식문서 소개글](https://modelcontextprotocol.io/introduction)  
- MCP는 MSA와 유사하게 클라이언트-서버 구조로 크게 나뉘어 있습니다.

- **클라이언트**는 Claude Desktop, Cursor 등에 탑재되어 있고, 직접 만들 수도 있습니다.
- **서버**는 이미 많은 수가 개발되어 있어 [MCP.so](https://mcp.so/)에서 원하는 것을 선택해 사용할 수 있습니다.
서버들은 외부 자원들 DB, SNS, 업무 툴(Slack, Github) 등 다양한 외부 자원과 연동할 수 있습니다. 저의 경우 사내 백오피스 툴과의 연동이 필요했기 때문에 커스텀 서버를 직접 개발했습니다.

### MCP Client & Server 아키텍쳐 이해
- 언어는 Python, TypeScript, Java, Kotlin, C# SDK를 제공하므로 선호하는 언어를 선택할 수 있습니다.
- 통신 방식: 크게 두 가지 통신 방식이 있습니다.
  - Standard Input/Output (STDIO): 
    - 로컬에서 Docker로 서버를 실행하고 Claude Desktop으로 클라이언트를 실행하는 방식에 주로 사용됩니다.
    - 간단하게 PC에서 클라이언트와 서버를 모두 실행하여 통신하는 형태입니다.
    - 모놀리식 서버 구조와 유사하다고 이해할 수 있습니다.
  - Streamable HTTP (SSE): 
    - 클라이언트와 서버가 분리되어 HTTP POST를 통해 통신하는 방식입니다.
    - 클라이언트와 서버가 분리되어 통신하는 형태입니다.
    - MSA 구조와 유사하다고 이해할 수 있습니다.
- 주요 기능
  - Resources: 서버가 클라이언트에 필요한 데이터나 콘텐츠를 공개하는 기능입니다.
  - Prompts: 재사용 가능한 프롬프트 템플릿이나 워크플로우를 정의하여 상호작용을 표준화하는 기능입니다.
  - Tools: 외부 서비스를 통해 정의된 작업들을 수행하는 기능입니다.
  
**여기서 궁금증이였던 Resources vs Tools**
- Tools: 클라이언트가 특정 행동 또는 추론을 수행할 때 선택적으로 사용하도록 돕는 도구입니다. 예를 들어, 특정 API를 호출하거나 데이터를 조회하는 등의 기능을 담당합니다.
- Resources: 클라이언트에게 필요한 데이터를 제공하여 컨텍스트 과부하를 방지하고, 클라이언트가 특정 정보에 한정적으로 접근하도록 합니다.

> 예를 들어, 공식 문서나 사용자 정보를 제공하는 것은 Resources의 역할이고, 문서 조회나 사용자에게 메시지 전송 등의 기능을 수행하는 것은 Tools의 역할이라고 이해하면 됩니다.


## SSE 기반 Server 작성
- 서버를 쿠버네티스 환경에 배포할 예정이었기 때문에 SSE 통신 방식을 선택했습니다. 그리고 헬스체크 엔드포인트를 필수로 추가했습니다.
- 대부분의 오픈소스 예제 코드는 STDIO 통신 형태의 서버가 많지만, 저는 SSE 통신 방식의 서버를 직접 개발했습니다.

### Python
[Python SDK Github](https://github.com/modelcontextprotocol/python-sdk)


#### 예제 코드
[전체 코드 확인](https://github.com/reumachoi/mcp-practice/blob/main/python-server-sse/main.py)
- Starlette를 사용하여 웹 애플리케이션의 라우팅을 정의하고, `/health` 엔드포인트와 MCP의 SSE 앱(`sse_app`)을 마운트합니다.
- Uvicorn을 통해 3000번 포트에서 서버를 실행합니다.
- URL: `http://{domain}:3000/sse`

```
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP
from starlette.applications import Starlette
from starlette.responses import PlainTextResponse
from starlette.routing import Mount, Route
import uvicorn 

# Initialize FastMCP server
mcp = FastMCP("weather")

def health_check(request):
    return PlainTextResponse('OK')

app = Starlette(
    routes=[
        Route('/health', health_check, methods=["GET"]),
        Mount('/', app=mcp.sse_app()),
    ]
)

# Constants
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"

@mcp.resource("echo://{message}")
def echo_resource(message: str) -> str:
    """Echo a message as a resource"""
    return f"Resource echo: {message}"


@mcp.tool()
def calculate_bmi(weightKg: int, heightM: int) -> str:
    """Echo a message as a tool"""
    return f"Tool echo: {(weightKg / (heightM * heightM))}"


@mcp.prompt()
def echo_prompt(message: str) -> str:
    """Create an echo prompt"""
    return f"Please process this message: {message}"


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=3000)
```


### TypeScript
[TypeScript SDK Github](https://github.com/modelcontextprotocol/typescript-sdk)

#### 예제코드
[전체 코드 확인](https://github.com/reumachoi/mcp-practice/blob/main/typescript-server-sse/src/index.ts)
- express 프레임워크를 사용해서 http 엔드포인트를 설정하고, `/sse`, `/messages` 엔드포인트로 SSE 통신을 처리합니다.
- sdk를 사용해서 McpServer 인스턴스를 생성하고, `SSEServerTransport` 설정을 합니다.
- URL: `http://{domain}:3000/sse`
  
```
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
// import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import { z } from "zod";
import express from "express";  

// Create server instance
const server = new McpServer({
  name: "my-ts-server",
  version: "1.0.0",
  capabilities: {
    tools: {},
  }
});

const app = express();
let transport: SSEServerTransport | null = null;

app.get("/sse", (req, res) => {
  transport = new SSEServerTransport("/messages", res);
  server.connect(transport);
});

app.post("/messages", (req, res) => {
  if (transport) {
    transport.handlePostMessage(req, res);
  }
});

app.get('/health', (req, res) => {
    res.json({ status: 'ok'});
 });

 server.tool(
    "calculate-bmi",
    {
      weightKg: z.number(),
      heightM: z.number()
    },
    async ({ weightKg, heightM }) => ({
      content: [{
        type: "text",
        text: String(weightKg / (heightM * heightM))
      }]
    })
  );

 
 app.listen(3000, async () => {
    console.log('Server is running on http://0.0.0.0:3000');
})

// Handle server shutdown
process.on('SIGINT', async () => {
    console.log('Shutting down server...');
    process.exit(0);
});
```

## 마치며
- 다양한 서버들이 빠르게 출시되고, 여러 툴에서 mcp를 지원하면서 전부 쫓아가기는 어려운 것 같습니다. 하지만 직접 만들어보고 개념에 대한 이해하면서 기술에 대한 장벽을 낮춰가고 다양한 방법으로 활용해볼 수 있는 기회가 되었습니다.

## 참고문서
- [MCP 공식 사이트](https://modelcontextprotocol.io/introduction)
- [MCP 공식 깃헙](https://github.com/modelcontextprotocol)