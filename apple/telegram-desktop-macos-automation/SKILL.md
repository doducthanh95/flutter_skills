---
name: telegram-desktop-macos-automation
description: Automate Telegram Desktop on macOS with AppleScript/System Events when browser/web login is unavailable. Use for activating the app, locating the current chat, typing into the message box, and sending a message.
version: 1.0.0
tags: [macos, telegram, automation, applescript, system-events, ui]
---

# Telegram Desktop macOS Automation

## Mục đích
Dùng skill này khi cần thao tác với Telegram Desktop trên macOS mà không dùng Telegram Web, đặc biệt khi:
- app Telegram đã mở sẵn trên máy
- cần kiểm tra đang ở chat/group nào
- cần nhắn tin vào group cá nhân hóa
- UI không có sẵn API hoặc browser session không đăng nhập

## Khi nào dùng
- User bảo “mở app Telegram trên máy”, “nhắn vào group Telegram”, “gửi tin trong Telegram desktop”
- Browser web Telegram đang ở login screen nhưng Telegram app native đã đăng nhập sẵn
- Cần tự động nhắn tin qua UI accessibility thay vì web

## Quy trình chuẩn

### 1) Kích hoạt Telegram Desktop
```bash
osascript -e 'tell application "Telegram" to activate'
```

Nếu cần xác nhận process:
```bash
osascript -e 'tell application "System Events" to get name of every process whose background only is false'
```

### 2) Xác định cửa sổ/chats đang mở
Dùng `System Events` để đọc `front window`:
```bash
osascript <<'APPLESCRIPT'
tell application "System Events"
  tell process "Telegram"
    set frontmost to true
    tell front window
      repeat with s in static texts
        try
          if (name of s as text) is not "" then log (name of s as text)
        end try
      end repeat
    end tell
  end tell
end tell
APPLESCRIPT
```

Thường chat title/group title sẽ xuất hiện trong `static texts` hoặc name của window.

### 3) Tìm ô nhập tin nhắn
Trong nhiều bản Telegram Desktop, ô nhập có thể được nhận diện như một `text field` với tên kiểu:
- `Write a message...`
- `Search`

Khám phá các text fields:
```bash
osascript <<'APPLESCRIPT'
tell application "System Events"
  tell process "Telegram"
    tell front window
      repeat with t in text fields
        try
          return (name of t as text) & "|" & (value of t as text)
        end try
      end repeat
    end tell
  end tell
end tell
APPLESCRIPT
```

### 4) Nhập nội dung và gửi
Ví dụ:
```bash
osascript <<'APPLESCRIPT'
tell application "System Events"
  tell process "Telegram"
    set frontmost to true
    tell front window
      repeat with t in text fields
        try
          if (name of t as text) contains "Write a message" then
            set value of t to "Hermes viết: Chúc mọi người cuối tuần vui vẻ nha!"
            exit repeat
          end if
        end try
      end repeat
    end tell
    keystroke return
  end tell
end tell
APPLESCRIPT
```

## Mẹo thực chiến
- Nếu set `value` không ăn, thử `keystroke` từng ký tự hoặc dùng `Cmd+V` với clipboard.
- Nếu focus chưa đúng, `tell application "Telegram" to activate` + `set frontmost to true` trước khi gõ.
- Nếu chat/group chưa mở đúng, dùng search field trong Telegram để tìm group trước, rồi mới nhập tin.
- Sau khi gửi, kiểm tra lại text field đã trống hoặc message đã xuất hiện trong chat history.

## Pitfalls thường gặp
- Telegram Web đang login nhưng Telegram Desktop vẫn mở sẵn: ưu tiên app native nếu user muốn thao tác trên máy tính.
- `value of text field` có thể trả về `missing value` dù vẫn có thể set được.
- Tên control thay đổi theo phiên bản Telegram; cần inspect lại `text fields`, `buttons`, `static texts`.
- Cửa sổ frontmost nhưng focus chưa ở ô nhập tin nhắn.

## Ghi chú từ thực nghiệm
- Trên máy này, `osascript` có thể thấy process `Telegram` và window title của nhóm hiện tại.
- Ô nhập tin nhắn có thể hiện dưới text field tên `Write a message...Search`.
- Có thể gửi message bằng cách set text field rồi `keystroke return`.
- Ví dụ group title đọc được qua Accessibility: `Chill cuối tuần`.
