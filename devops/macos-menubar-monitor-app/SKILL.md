---
name: macos-menubar-monitor-app
description: Tạo macOS menu bar app để monitor CLI tools realtime (config, logs, metrics) bằng Python rumps
tags: [macos, menubar, monitoring, python, rumps, realtime]
---

# macOS Menu Bar Monitor App

Tạo ứng dụng chạy trên menu bar của macOS để monitor CLI tools realtime - hiển thị config, parse logs, và tính metrics (token usage, cost, rate).

## Khi nào dùng

- Monitor AI agent (provider, model, token usage, cost)
- Track CLI tool metrics realtime từ config + log files
- Cần UI nhẹ, luôn hiển thị trên menu bar
- Muốn notification khi có thay đổi quan trọng

## Dependencies

```bash
pip3 install --user rumps pyyaml
```

**Lưu ý**: 
- `rumps` requires PyObjC (tự động cài khi install rumps)
- Chỉ chạy trên macOS (requires Cocoa framework)
- Nếu `pip3 install` timeout, dùng `--user` flag

## Template cơ bản

```python
#!/usr/bin/env python3
import rumps
import yaml
from pathlib import Path
from datetime import datetime

class MyMonitor(rumps.App):
    def __init__(self):
        super(MyMonitor, self).__init__("🔍 Monitor", quit_button="Thoát")
        
        # Paths to config and logs
        self.config_path = Path.home() / ".tool" / "config.yaml"
        self.log_path = Path.home() / ".tool" / "logs" / "app.log"
        
        # State variables
        self.metric1 = 0
        self.metric2 = "N/A"
        
        # Menu items
        self.menu_item1 = rumps.MenuItem(f"Metric 1: {self.metric1}")
        self.menu_item2 = rumps.MenuItem(f"Metric 2: {self.metric2}")
        self.menu_refresh = rumps.MenuItem("🔄 Làm mới", callback=self.refresh)
        
        self.menu = [
            self.menu_item1,
            self.menu_item2,
            rumps.separator,
            self.menu_refresh
        ]
        
        # Auto-update timer (every N seconds)
        self.timer = rumps.Timer(self.update_stats, 2)
        self.timer.start()
    
    def load_config(self):
        """Parse YAML config"""
        if self.config_path.exists():
            with open(self.config_path, 'r') as f:
                config = yaml.safe_load(f)
                self.metric2 = config.get('key', 'N/A')
    
    def parse_logs(self):
        """Parse log file cho metrics"""
        if not self.log_path.exists():
            return
        
        with open(self.log_path, 'r') as f:
            lines = f.readlines()[-1000:]  # 1000 dòng cuối
        
        # Regex parsing logic here
        import re
        for line in lines:
            match = re.search(r'pattern: (\d+)', line)
            if match:
                self.metric1 = int(match.group(1))
    
    def update_stats(self, _):
        """Callback tự động mỗi N giây"""
        self.load_config()
        self.parse_logs()
        
        # Update menu items
        self.menu_item1.title = f"Metric 1: {self.metric1}"
        self.menu_item2.title = f"Metric 2: {self.metric2}"
        
        # Update title bar
        self.title = f"🔍 {self.metric1}"
    
    @rumps.clicked("🔄 Làm mới")
    def refresh(self, _):
        self.update_stats(None)
        rumps.notification(
            title="Monitor",
            subtitle="Đã làm mới",
            message=f"Metric: {self.metric1}"
        )

if __name__ == "__main__":
    app = MyMonitor()
    app.run()
```

## Ví dụ: AI Agent Monitor (Hermes) - SQLite DB Version

Monitor provider, model, token usage từ Hermes config + SQLite DB:

```python
#!/usr/bin/env python3
import rumps
import yaml
import sqlite3
from pathlib import Path
from datetime import datetime

class AIMonitor(rumps.App):
    def __init__(self):
        super(AIMonitor, self).__init__("🤖 AI", quit_button="Thoát")
        
        self.config_path = Path.home() / ".hermes" / "config.yaml"
        self.db_path = Path.home() / ".hermes" / "state.db"
        
        self.provider = "N/A"
        self.model = "N/A"
        self.tokens_in = 0
        self.tokens_out = 0
        self.tokens_per_min = 0
        self.cost = 0.0
        
        self.menu_provider = rumps.MenuItem(f"Provider: {self.provider}")
        self.menu_model = rumps.MenuItem(f"Model: {self.model}")
        self.menu_tokens_in = rumps.MenuItem(f"Tokens In: {self.tokens_in:,}")
        self.menu_tokens_out = rumps.MenuItem(f"Tokens Out: {self.tokens_out:,}")
        self.menu_rate = rumps.MenuItem(f"Rate: {self.tokens_per_min} tok/min")
        self.menu_cost = rumps.MenuItem(f"Cost: ${self.cost:.4f}")
        
        self.menu = [
            self.menu_provider,
            self.menu_model,
            rumps.separator,
            self.menu_tokens_in,
            self.menu_tokens_out,
            self.menu_rate,
            self.menu_cost,
        ]
        
        self.timer = rumps.Timer(self.update_stats, 2)
        self.timer.start()
    
    def load_config(self):
        if self.config_path.exists():
            with open(self.config_path, 'r') as f:
                config = yaml.safe_load(f)
                model_config = config.get('model', {})
                self.provider = model_config.get('provider', 'N/A')
                self.model = model_config.get('default', 'N/A')
    
    def parse_db_for_tokens(self):
        """Parse SQLite DB để lấy token usage"""
        if not self.db_path.exists():
            return
        
        try:
            conn = sqlite3.connect(str(self.db_path))
            cursor = conn.cursor()
            
            # Query sessions từ 1 giờ trước
            query = """
                SELECT 
                    json_extract(data, '$.token_usage.input_tokens') as tokens_in,
                    json_extract(data, '$.token_usage.output_tokens') as tokens_out,
                    updated_at
                FROM sessions 
                WHERE updated_at >= datetime('now', '-1 hour')
                ORDER BY updated_at DESC
                LIMIT 100
            """
            
            cursor.execute(query)
            rows = cursor.fetchall()
            
            total_in = 0
            total_out = 0
            timestamps = []
            
            for row in rows:
                if row[0]: total_in += int(row[0])
                if row[1]: total_out += int(row[1])
                if row[2]: timestamps.append(row[2])
            
            if total_in > 0:
                self.tokens_in = total_in
                self.tokens_out = total_out
                
                # Tính tốc độ
                if len(timestamps) >= 2:
                    first = datetime.fromisoformat(timestamps[-1])
                    last = datetime.fromisoformat(timestamps[0])
                    duration_min = (last - first).total_seconds() / 60
                    if duration_min > 0:
                        self.tokens_per_min = int((total_in + total_out) / duration_min)
                
                # Cost estimate (Claude Sonnet 4.5: $3/M in, $15/M out)
                self.cost = (total_in / 1_000_000) * 3.0 + (total_out / 1_000_000) * 15.0
            
            conn.close()
        except Exception as e:
            print(f"DB error: {e}")
    
    def update_stats(self, _):
        self.load_config()
        self.parse_db_for_tokens()
        
        self.menu_provider.title = f"Provider: {self.provider}"
        self.menu_model.title = f"Model: {self.model}"
        self.menu_tokens_in.title = f"Tokens In: {self.tokens_in:,}"
        self.menu_tokens_out.title = f"Tokens Out: {self.tokens_out:,}"
        self.menu_rate.title = f"Rate: {self.tokens_per_min:,} tok/min"
        self.menu_cost.title = f"Cost: ${self.cost:.4f}"
        
        # Update title bar
        provider_short = {
            "anthropic": "Claude",
            "openai": "GPT",
            "google": "Gemini"
        }.get(self.provider, self.provider)
        
        if self.tokens_in > 0:
            self.title = f"🤖 {provider_short} | {self.tokens_in:,}"
        else:
            self.title = f"🤖 {provider_short}"

if __name__ == "__main__":
    app = AIMonitor()
    app.run()
```

**Auto-start/stop với Hermes**:

Tạo scripts:
```bash
# ~/.hermes/scripts/start_monitor.sh
#!/bin/bash
MONITOR_SCRIPT="$HOME/ai_monitor.py"
MONITOR_PID_FILE="$HOME/.hermes/monitor.pid"

if [ -f "$MONITOR_PID_FILE" ]; then
    OLD_PID=$(cat "$MONITOR_PID_FILE")
    if ps -p "$OLD_PID" > /dev/null 2>&1; then
        exit 0
    fi
fi

python3 "$MONITOR_SCRIPT" &
echo $! > "$MONITOR_PID_FILE"
```

```bash
# ~/.hermes/scripts/stop_monitor.sh
#!/bin/bash
MONITOR_PID_FILE="$HOME/.hermes/monitor.pid"

if [ -f "$MONITOR_PID_FILE" ]; then
    kill $(cat "$MONITOR_PID_FILE") 2>/dev/null
    rm "$MONITOR_PID_FILE"
fi
```

Thêm vào `~/.zshrc`:
```bash
function hermes() {
    local HERMES_BIN="$HOME/.local/bin/hermes"
    bash "$HOME/.hermes/scripts/start_monitor.sh"
    "$HERMES_BIN" "$@"
    bash "$HOME/.hermes/scripts/stop_monitor.sh"
}
```

Reload: `source ~/.zshrc`

**Chạy:**
```bash
chmod +x ~/ai_monitor.py
python3 ~/ai_monitor.py &
```

Icon 🤖 sẽ xuất hiện trên menu bar.

## Chạy background

**Option 1: Terminal background**
```bash
python3 ~/monitor.py &
```

**Option 2: LaunchAgent (tự động khởi động khi login)**

Tạo `~/Library/LaunchAgents/com.user.monitor.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.monitor</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/Users/YOUR_USERNAME/ai_monitor.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

Load:
```bash
launchctl load ~/Library/LaunchAgents/com.user.monitor.plist
```

Unload:
```bash
launchctl unload ~/Library/LaunchAgents/com.user.monitor.plist
```

## Pitfalls

1. **pip install timeout**: Nếu `pip3 install rumps` timeout, dùng `--user` flag hoặc tăng timeout
2. **Permission denied**: App cần quyền accessibility nếu muốn monitor system events - vào System Preferences → Security & Privacy → Accessibility
3. **Icon không hiện**: Check process có đang chạy không (`ps aux | grep monitor.py`)
4. **Log file quá lớn**: Chỉ parse N dòng cuối (vd: `lines[-1000:]`) để tránh OOM
5. **YAML parsing error**: Luôn wrap trong try-except khi parse config
6. **Datetime parsing**: Đảm bảo format string khớp với log format
7. **Hermes token usage location**: Token usage KHÔNG nằm trong plaintext logs (`~/.hermes/logs/`). Thay vào đó, parse từ **SQLite DB** (`~/.hermes/state.db`) bảng `sessions` với `json_extract(data, '$.token_usage.input_tokens')` hoặc từ session JSON files trong `~/.hermes/sessions/`

## Customization

**Thay đổi update frequency:**
```python
self.timer = rumps.Timer(self.update_stats, 5)  # 5 giây
```

**Thêm notification khi metric vượt threshold:**
```python
def update_stats(self, _):
    # ... update logic ...
    
    if self.tokens_per_min > 10000:
        rumps.notification(
            title="Cảnh báo!",
            subtitle="Token rate cao",
            message=f"{self.tokens_per_min} tok/min"
        )
```

**Dynamic icon dựa trên state:**
```python
if self.provider == "anthropic":
    self.title = "🤖 Claude"
elif self.provider == "openai":
    self.title = "🤖 GPT"
else:
    self.title = f"🤖 {self.provider}"
```

## Verification

1. **Check icon xuất hiện**: Icon phải visible trên menu bar (góc phải, gần clock)
2. **Click icon**: Menu dropdown phải hiển thị metrics
3. **Auto-update**: Metrics phải tự động cập nhật mỗi N giây
4. **Manual refresh**: Click "🔄 Làm mới" phải trigger update ngay lập tức
5. **Notification**: Phải nhận notification khi click refresh

## Related Skills

- `apple-*` skills: macOS automation workflows
- `copilot-debugging-workflow`: Debugging process monitoring
- `systematic-debugging`: Log parsing techniques
