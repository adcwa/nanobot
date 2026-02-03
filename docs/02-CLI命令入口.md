# 第一层：CLI 命令入口

> 📌 **核心文件**：`nanobot/cli/commands.py` (~656 行)

## 概述

CLI（Command Line Interface）是 nanobot 的主要用户接口，基于 [Typer](https://typer.tiangolo.com/) 框架构建。所有命令都在一个文件中定义，简洁而强大。

## Typer 框架简介

Typer 是一个现代的 Python CLI 框架，特点：
- 自动生成帮助文档
- 类型提示自动验证
- 子命令支持
- 交互式提示

```python
import typer

app = typer.Typer(
    name="nanobot",
    help="🐈 nanobot - Personal AI Assistant",
    no_args_is_help=True,  # 无参数时显示帮助
)

@app.command()
def hello(name: str = typer.Option("World", help="Name to greet")):
    """Say hello"""
    print(f"Hello {name}!")

if __name__ == "__main__":
    app()
```

## 核心命令详解

### 1. `onboard` - 初始化

**用途**：创建配置文件和工作区

```python
@app.command()
def onboard():
    """Initialize nanobot configuration and workspace."""
    config_dir = Path.home() / ".nanobot"
    config_file = config_dir / "config.json"
    
    # 创建目录结构
    config_dir.mkdir(exist_ok=True)
    (config_dir / "sessions").mkdir(exist_ok=True)
    (config_dir / "memory").mkdir(exist_ok=True)
    (config_dir / "skills").mkdir(exist_ok=True)
    
    # 创建默认配置
    if not config_file.exists():
        default_config = {
            "providers": {
                "openrouter": {"apiKey": ""}
            },
            "agents": {
                "defaults": {"model": "anthropic/claude-opus-4-5"}
            }
        }
        config_file.write_text(json.dumps(default_config, indent=2))
    
    # 创建工作区模板
    _create_workspace_templates(config_dir)
    
    console.print("✓ Configuration initialized at ~/.nanobot")
```

**创建的文件**：
```
~/.nanobot/
├── config.json       # 主配置文件
├── sessions/         # 会话历史
├── memory/           # 记忆存储
│   └── MEMORY.md
├── skills/           # 自定义技能
├── AGENTS.md         # Agent 行为规范
├── SOUL.md           # 个性化设定
└── USER.md           # 用户信息
```

### 2. `agent` - 与 Agent 交互

**用途**：直接与 Agent 对话

```python
@app.command()
def agent(
    message: str = typer.Option(None, "--message", "-m", help="Message to send"),
    session_id: str = typer.Option("cli:default", "--session", "-s", help="Session ID"),
):
    """Interact with the agent directly."""
    
    # 加载配置
    config = ConfigLoader().load()
    
    # 创建 Agent
    provider = create_provider(config)
    agent_loop = AgentLoop(
        bus=MessageBus(),  # CLI 模式不需要真正的消息总线
        provider=provider,
        workspace=Path.home() / ".nanobot",
        model=config.agents.defaults.model
    )
    
    if message:
        # 单次对话模式
        response = asyncio.run(agent_loop.process_direct(message, session_id))
        console.print(response)
    else:
        # 交互式模式
        while True:
            user_input = Prompt.ask("[bold blue]You[/bold blue]")
            if user_input.lower() in ["exit", "quit"]:
                break
            
            response = asyncio.run(agent_loop.process_direct(user_input, session_id))
            console.print(f"[bold green]Agent[/bold green]: {response}")
```

**使用方式**：

```bash
# 单次对话
nanobot agent -m "你好"
nanobot agent --message "读取 README.md 文件"

# 交互式对话
nanobot agent
You: 你好
Agent: 你好！我是 nanobot，有什么可以帮助你的？
You: 退出
```

### 3. `gateway` - 启动消息网关

**用途**：支持多渠道（Telegram/WhatsApp）的后台服务

```python
@app.command()
def gateway(
    port: int = typer.Option(18790, "--port", "-p", help="Gateway port"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Verbose output"),
):
    """Start the nanobot gateway."""
    
    # 设置日志级别
    if verbose:
        logger.remove()
        logger.add(sys.stderr, level="DEBUG")
    
    # 加载配置
    config = ConfigLoader().load()
    workspace = Path.home() / ".nanobot"
    
    # 创建核心组件
    bus = MessageBus()
    provider = create_provider(config)
    
    # Agent Loop
    agent_loop = AgentLoop(
        bus=bus,
        provider=provider,
        workspace=workspace,
        model=config.agents.defaults.model,
        brave_api_key=config.tools.web.search.apiKey if config.tools.web.search else None
    )
    
    # 渠道管理器
    channel_manager = ChannelManager(config, bus)
    
    # Cron 服务
    cron_service = CronService(workspace / "cron.json")
    
    # 定义 Cron 回调
    async def on_cron_job(job: CronJob):
        """执行定时任务"""
        await bus.publish_inbound(InboundMessage(
            channel=job.channel or "cli",
            sender_id="cron",
            chat_id=job.to or "system",
            content=job.message,
            session_key=f"cron:{job.id}"
        ))
    
    cron_service.set_callback(on_cron_job)
    
    # 运行所有服务
    async def run():
        await asyncio.gather(
            agent_loop.run(),           # Agent 主循环
            bus.dispatch_outbound(),    # 消息分发
            channel_manager.start(),    # 渠道监听
            cron_service.start(),       # 定时任务
        )
    
    console.print(f"🚀 nanobot gateway started on port {port}")
    asyncio.run(run())
```

**服务架构**：

```
┌─────────────────┐
│  Telegram Bot   │──┐
└─────────────────┘  │
                     │
┌─────────────────┐  │    ┌──────────────┐
│  WhatsApp       │──┼───→│ MessageBus   │
└─────────────────┘  │    └──────┬───────┘
                     │           │
┌─────────────────┐  │           ▼
│  Cron Service   │──┘    ┌──────────────┐
└─────────────────┘       │ AgentLoop    │
                          └──────────────┘
```

### 4. `cron` - 定时任务管理

**子命令结构**：

```python
cron_app = typer.Typer(help="Manage scheduled tasks")
app.add_typer(cron_app, name="cron")
```

#### 4.1 `cron list` - 列出任务

```bash
nanobot cron list
nanobot cron list --all  # 包括禁用的任务
```

#### 4.2 `cron add` - 添加任务

```python
@cron_app.command("add")
def cron_add(
    name: str = typer.Option(..., "--name", "-n"),
    message: str = typer.Option(..., "--message", "-m"),
    cron: str = typer.Option(None, "--cron"),
    every: int = typer.Option(None, "--every"),
    to: str = typer.Option(None, "--to"),
    channel: str = typer.Option(None, "--channel"),
):
    """Add a scheduled job."""
    
    # 验证：必须指定 cron 或 every
    if not cron and not every:
        console.print("[red]Error: Must specify either --cron or --every[/red]")
        raise typer.Exit(1)
    
    # 创建任务
    job = CronJob(
        id=str(uuid.uuid4()),
        name=name,
        message=message,
        cron_expr=cron,
        interval_seconds=every,
        to=to,
        channel=channel,
        enabled=True
    )
    
    service.add_job(job)
    console.print(f"✓ Added job {job.id}")
```

**使用示例**：

```bash
# 使用 cron 表达式（每天 9 点）
nanobot cron add --name "morning" --message "早上好！" --cron "0 9 * * *"

# 使用间隔秒数（每小时）
nanobot cron add --name "hourly" --message "检查状态" --every 3600

# 指定接收者
nanobot cron add --name "reminder" \
  --message "记得喝水" \
  --cron "0 */2 * * *" \
  --channel "telegram" \
  --to "123456789"
```

#### 4.3 `cron remove` - 删除任务

```bash
nanobot cron remove <job_id>
```

#### 4.4 `cron enable/disable` - 启用/禁用

```bash
nanobot cron enable <job_id>
nanobot cron enable <job_id> --disable
```

### 5. `channels` - 渠道管理

#### 5.1 `channels status` - 查看状态

```python
@channels_app.command("status")
def channels_status():
    """Show channel status."""
    config = ConfigLoader().load()
    
    table = Table(title="Channel Status")
    table.add_column("Channel", style="cyan")
    table.add_column("Enabled", style="green")
    table.add_column("Config", style="yellow")
    
    # Telegram
    if config.channels.telegram:
        table.add_row(
            "Telegram",
            "✓" if config.channels.telegram.enabled else "✗",
            f"Token: {config.channels.telegram.token[:10]}..."
        )
    
    # WhatsApp
    if config.channels.whatsapp:
        table.add_row(
            "WhatsApp",
            "✓" if config.channels.whatsapp.enabled else "✗",
            f"Allowed: {len(config.channels.whatsapp.allowFrom)} users"
        )
    
    console.print(table)
```

#### 5.2 `channels login` - WhatsApp 登录

```bash
nanobot channels login
# 显示二维码，用 WhatsApp 扫描
```

### 6. `status` - 系统状态

```python
@app.command()
def status():
    """Show nanobot status."""
    config_dir = Path.home() / ".nanobot"
    config_file = config_dir / "config.json"
    
    # 基本信息
    console.print(f"[bold]nanobot Status[/bold]")
    console.print(f"Version: {__version__}")
    console.print(f"Config: {config_file}")
    console.print(f"Workspace: {config_dir}")
    
    # 配置状态
    if config_file.exists():
        config = ConfigLoader().load()
        console.print(f"Model: {config.agents.defaults.model}")
        console.print(f"Providers: {', '.join(config.providers.keys())}")
    
    # 会话统计
    sessions = list((config_dir / "sessions").glob("*.json"))
    console.print(f"Sessions: {len(sessions)}")
    
    # Cron 任务
    cron_file = config_dir / "cron.json"
    if cron_file.exists():
        jobs = json.loads(cron_file.read_text())
        console.print(f"Cron jobs: {len(jobs)}")
```

## 命令的组织结构

```python
app (主应用)
├── onboard()          # 初始化
├── agent()            # Agent 交互
├── gateway()          # 启动网关
├── status()           # 系统状态
└── 子应用
    ├── cron_app
    │   ├── list()
    │   ├── add()
    │   ├── remove()
    │   ├── enable()
    │   └── run()
    └── channels_app
        ├── status()
        └── login()
```

## 配置加载流程

每个命令都需要加载配置：

```python
from nanobot.config.loader import ConfigLoader

def load_config():
    try:
        config = ConfigLoader().load()
        return config
    except FileNotFoundError:
        console.print("[red]Config not found. Run 'nanobot onboard' first.[/red]")
        raise typer.Exit(1)
    except Exception as e:
        console.print(f"[red]Error loading config: {e}[/red]")
        raise typer.Exit(1)
```

## Rich 输出样式

nanobot 使用 Rich 库美化输出：

```python
from rich.console import Console
from rich.table import Table
from rich.panel import Panel

console = Console()

# 表格
table = Table(title="Results")
table.add_column("Name", style="cyan")
table.add_column("Value", style="green")
table.add_row("Key", "Value")
console.print(table)

# 面板
panel = Panel("Important message", title="Info", border_style="blue")
console.print(panel)

# 进度条
from rich.progress import track
for i in track(range(100), description="Processing..."):
    # 工作
    pass
```

## 错误处理

```python
try:
    # 操作
    result = do_something()
except ConfigError as e:
    console.print(f"[red]Config error: {e}[/red]")
    raise typer.Exit(1)
except Exception as e:
    logger.exception("Unexpected error")
    console.print(f"[red]Error: {e}[/red]")
    raise typer.Exit(1)
```

## 交互式提示

```python
from rich.prompt import Prompt, Confirm

# 文本输入
name = Prompt.ask("Enter your name")

# 带默认值
model = Prompt.ask("Model", default="claude-opus-4-5")

# 确认
if Confirm.ask("Continue?"):
    proceed()
```

## 小结

通过本章，你应该了解了：
- ✅ Typer 框架的基本用法
- ✅ nanobot 的所有 CLI 命令
- ✅ 命令的实现细节
- ✅ 配置加载和错误处理
- ✅ Rich 库的输出美化

**下一步**：[03-配置系统.md](./03-配置系统.md) - 深入了解配置管理。
