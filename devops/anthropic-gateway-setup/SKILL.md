---
name: anthropic-gateway-setup
description: Setup Anthropic API key từ third-party gateway (gwai.cloud) cho Hermes Agent và Claude Code CLI
tags: [hermes, anthropic, claude-code, gateway, api-key, setup]
version: 1.0.0
---

# Anthropic Gateway Setup

Setup API key từ third-party gateway (như gwai.cloud) cho Hermes Agent và Claude Code CLI.

## When to Use

- Người dùng có API key từ gwai.cloud hoặc gateway tương tự (không phải từ console.anthropic.com)
- Cần cài đặt Claude Code CLI với custom gateway
- Cần cấu hình Hermes để sử dụng Anthropic gateway thay vì API chính thức
- API key có format `sk-ant-api03-...` nhưng không hoạt động với api.anthropic.com

## Key Indicators

- Người dùng cung cấp hướng dẫn install script từ gwai.cloud
- Script install chứa: `curl -fsSL https://1gw.gwai.cloud/setup-claude/install.sh`
- API key test với api.anthropic.com báo lỗi "invalid x-api-key"
- Có mention về gateway/proxy thay vì Anthropic trực tiếp

## Prerequisites

- macOS hoặc Linux
- curl và bash
- Hermes Agent đã cài đặt

## Implementation Steps

### 1. Install Claude Code CLI với Gateway

```bash
# Chạy install script từ gwai.cloud
curl -fsSL https://1gw.gwai.cloud/setup-claude/install.sh | bash -s -- --key <API_KEY>

# Verify installation
which claude
claude --version

# Kiểm tra config
cat ~/.claude/settings.json
```

**Expected output:**
- Claude Code CLI được cài đặt
- Settings file tại `~/.claude/settings.json` chứa:
  - `ANTHROPIC_AUTH_TOKEN`: API key
  - `ANTHROPIC_BASE_URL`: https://1gw.gwai.cloud
  - Các model defaults

### 2. Test Claude Code CLI

```bash
# Test với model sonnet
echo "Xin chào, trả lời bằng tiếng Việt" | claude --model sonnet

# Test với model opus
echo "Test connection" | claude --model opus
```

**Expected:** Claude trả lời thành công qua gateway.

### 3. Configure Hermes Agent

Update `~/.hermes/config.yaml`:

```yaml
model:
  default: claude-opus-4-6
  provider: anthropic
  api_mode: chat_completions
  base_url: https://1gw.gwai.cloud
providers:
  anthropic:
    base_url: https://1gw.gwai.cloud
    api_key: <API_KEY>
```

**Sử dụng patch tool:**

```bash
# Read current config first
read_file ~/.hermes/config.yaml

# Patch với base_url và api_key
patch mode=replace path=~/.hermes/config.yaml \
  old_string='model:
  default: claude-opus-4-6
  provider: anthropic
  api_mode: chat_completions
providers: {}' \
  new_string='model:
  default: claude-opus-4-6
  provider: anthropic
  api_mode: chat_completions
  base_url: https://1gw.gwai.cloud
providers:
  anthropic:
    base_url: https://1gw.gwai.cloud
    api_key: <API_KEY>'
```

### 4. Add API Key to Environment (Optional)

```bash
# Kiểm tra shell type
echo $SHELL

# Nếu zsh (macOS default)
echo 'export ANTHROPIC_API_KEY="<API_KEY>"' >> ~/.zshrc
source ~/.zshrc

# Nếu bash
echo 'export ANTHROPIC_API_KEY="<API_KEY>"' >> ~/.bashrc
source ~/.bashrc
```

### 5. Test Gateway Connection

```bash
# Liệt kê models có sẵn trên gateway
curl -s https://1gw.gwai.cloud/v1/models \
  -H "x-api-key: <API_KEY>" | python3 -m json.tool

# Test kết nối với model cụ thể
curl -s https://1gw.gwai.cloud/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: <API_KEY>" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-6",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

**Expected:** 
- List models: JSON array với các models available (có thể có prefix `cdd/`, `viber/`, `kiro-`)
- Test connection: JSON response với `"role":"assistant"` và content
- Nếu lỗi `"model_not_allowed"`: API key không được phép dùng model đó

### 6. Restart Hermes

```bash
# Exit current session
exit

# Restart Hermes để load config mới
hermes
```

## Pitfalls & Troubleshooting

### ❌ GitHub Copilot premium request tăng lên khi setup Anthropic gateway

**Symptom:** Sau khi cấu hình Anthropic API key, số lượng GitHub Copilot premium request vẫn tăng lên.

**Cause:** 
- API key bị lỗi (403/401) khi Hermes thử kết nối
- Hermes tự động fallback sang GitHub Copilot provider
- User không nhận ra vì conversation vẫn hoạt động bình thường

**Diagnosis:**
```bash
# Kiểm tra logs gần đây
tail -100 ~/.hermes/logs/agent.log | grep -i "provider\|copilot\|anthropic\|model"

# Tìm các dấu hiệu:
# - "credential pool: marking ANTHROPIC_API_KEY exhausted (status=403)"
# - "Model switched in-place: ... (anthropic) -> ... (github-copilot)"
# - "WARNING: anthropic requested but no Anthropic credentials found"
```

**Solution:**
1. Test API key với gateway URL (KHÔNG phải api.anthropic.com):
   ```bash
   curl -s -w "\nHTTP_CODE:%{http_code}\n" https://1gw.gwai.cloud/v1/messages \
     -H "content-type: application/json" \
     -H "x-api-key: <API_KEY>" \
     -H "anthropic-version: 2023-06-01" \
     -d '{"model": "claude-opus-4-6", "max_tokens": 10, "messages": [{"role": "user", "content": "Hi"}]}'
   ```

2. Nếu thấy `"model_not_allowed"`, liên hệ gwai.cloud để:
   - Xác nhận models nào được phép với key này
   - Kích hoạt thêm models

3. Nếu key hợp lệ, update config và restart Hermes:
   ```bash
   # Restart để load config mới
   pkill -f hermes
   hermes
   ```

4. Nếu không thể fix, xem xét chỉ dùng GitHub Copilot làm provider chính:
   ```bash
   # Xóa provider anthropic khỏi config
   patch mode=replace path=~/.hermes/config.yaml \
     old_string='providers:\n  anthropic:\n    base_url: https://1gw.gwai.cloud\n    api_key: ...' \
     new_string='providers: {}'
   ```

### ❌ `hermes --provider` không phải flag hợp lệ

**Symptom:** `hermes: error: argument command: invalid choice: 'anthropic'`

**Cause:** Hermes CLI không có flag `--provider`. Đây không phải cách test.

**Solution:** Test kết nối bằng curl trực tiếp đến gateway, hoặc dùng `hermes status` / `hermes doctor`.

### ❌ `hermes doctor` báo "invalid API key" dù gateway hoạt động bình thường

**Symptom:** `✗ Anthropic API (invalid API key)` trong hermes doctor.

**Cause:** `hermes doctor` hardcode test với `api.anthropic.com` chính thức, không dùng gateway URL. Đây là false negative — không phản ánh thực tế.

**Solution:** Bỏ qua cảnh báo này nếu curl đến gateway trả về 200. Dùng `hermes status` để xác nhận key được nhận diện đúng (`Anthropic ✓ sk-a...`).

### ❌ `.env` file bị protected, không thể patch trực tiếp

**Symptom:** `Write denied: '.env' is a protected system/credential file.`

**Cause:** Hermes bảo vệ file `.env` khỏi bị ghi đè.

**Solution:** Dùng `hermes config set` thay vì patch file:
```bash
hermes config set providers.anthropic.base_url https://1gw.gwai.cloud
hermes config set model.base_url https://1gw.gwai.cloud
```

### ❌ Testing với api.anthropic.com thay vì gateway

**Symptom:** Lỗi `"authentication_error": "invalid x-api-key"`

**Cause:** API key từ gwai.cloud chỉ hoạt động với gateway của họ, KHÔNG hoạt động với Anthropic API chính thức.

**Solution:** Luôn test với base URL của gateway:
```bash
# ❌ WRONG - sẽ fail
curl https://api.anthropic.com/v1/messages -H "x-api-key: $KEY" ...

# ✅ CORRECT
curl https://1gw.gwai.cloud/v1/messages -H "x-api-key: $KEY" ...
```

### ❌ Quên thêm base_url vào Hermes config

**Symptom:** Hermes không kết nối được hoặc báo authentication error.

**Cause:** Hermes mặc định dùng api.anthropic.com nếu không có base_url.

**Solution:** Phải thêm `base_url` ở 2 nơi trong config.yaml:
1. Trong section `model:`
2. Trong section `providers.anthropic:`

### ❌ Environment variable không persist

**Symptom:** API key bị mất sau khi restart terminal.

**Cause:** Chỉ export trong session hiện tại mà không lưu vào shell profile.

**Solution:** Thêm vào ~/.zshrc hoặc ~/.bashrc để persist.

### ❌ Source ~/.zshrc trong bash context

**Symptom:** Lỗi về Oh-My-Zsh, p10k, local, builtin...

**Cause:** Terminal tool của Hermes chạy trong bash, không thể source zsh config.

**Solution:** Không cần source trong terminal tool. Config sẽ tự động load khi mở terminal mới hoặc restart Hermes.

## Available Models

Gateway gwai.cloud hỗ trợ các models với nhiều tiers khác nhau:

**Standard tier (không prefix):**
- `claude-opus-4-6` - Mạnh nhất, phù hợp công việc phức tạp
- `claude-sonnet-4-6` - Cân bằng giữa tốc độ và chất lượng
- `claude-haiku-4-5` - Nhanh nhất, phù hợp task đơn giản

**Premium tiers (có thể yêu cầu API key đặc biệt):**
- `cdd/claude-opus-4.6`, `cdd/claude-sonnet-4.5` - CDD tier
- `viber/claude-opus-4.6`, `viber/claude-sonnet-4.5` - Viber tier
- `kiro-claude-sonnet-4-5`, `kiro-claude-haiku-4-5` - AWS Kiro tier

**Lưu ý:** Không phải API key nào cũng được phép dùng tất cả models. Nếu gặp lỗi `"model_not_allowed"`, liên hệ gwai.cloud để:
- Xác nhận tier của API key
- Upgrade hoặc kích hoạt thêm models
- Lấy danh sách chính xác models được phép dùng

## Verification Checklist

- [ ] Claude Code CLI đã cài đặt và test thành công
- [ ] ~/.claude/settings.json chứa đúng base_url và api_key
- [ ] ~/.hermes/config.yaml đã update base_url và api_key
- [ ] Test curl với gateway URL thành công (HTTP 200, không phải 403/401)
- [ ] Liệt kê models available và confirm key được phép dùng model mong muốn
- [ ] API key đã thêm vào ~/.zshrc hoặc ~/.bashrc (optional)
- [ ] Hermes restart và kết nối thành công
- [ ] **Kiểm tra logs không có fallback sang GitHub Copilot:** `tail ~/.hermes/logs/agent.log | grep -i copilot` không thấy "Model switched"

## Notes

- Gateway của gwai.cloud có thể có rate limits hoặc quota khác với Anthropic chính thức
- Không phải tất cả features/models của Anthropic đều có trên gateway
- Nếu gateway down, Hermes sẽ không thể kết nối - có thể setup fallback provider
- API key format: `sk-ant-api03-...` (99 ký tự)

## Related

- `claude-code` skill - Delegate tasks to Claude Code CLI
- `hermes-agent` skill - Complete guide to Hermes Agent
