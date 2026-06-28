# WebRTC P2P Webcam App — Spec cho Claude Code

## Tổng quan

Ứng dụng video call P2P thuần tĩnh (HTML + CSS + JS), không cần backend.  
Deploy trên **GitHub Pages** (HTTPS tự động → không bị block).  
Kết nối 2 browser trong cùng LAN hoặc khác mạng qua **PeerJS** (signaling) + **WebRTC** (media stream).

---

## Kiến trúc

```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Pages (HTTPS)                  │
│                     index.html (static)                  │
└──────────────┬─────────────────────────┬────────────────┘
               │ signaling only          │ signaling only
               ▼                         ▼
        ┌─────────────┐           ┌─────────────┐
        │  Browser A  │◄─────────►│  Browser B  │
        │  webcam     │  P2P RTP  │  webcam     │
        │  mic        │  (direct) │  mic        │
        └─────────────┘           └─────────────┘
               │                         │
               └──────────┬──────────────┘
                           ▼
                   PeerJS cloud server
                   (chỉ trao đổi SDP/ICE
                    — không có media)
```

---

## Cấu trúc file

```
/
├── index.html        ← toàn bộ app (1 file duy nhất)
└── README.md         ← hướng dẫn dùng
```

> Không cần build tool, không cần npm, không cần server.  
> Tất cả dependency load qua CDN (`unpkg.com`).

---

## Yêu cầu kỹ thuật

### Dependencies (CDN)
```html
<script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
```
Chỉ cần 1 thư viện này. Không cần thêm gì khác.

### Trình duyệt hỗ trợ
- Chrome 80+
- Firefox 102+
- Edge 80+
- Safari 14+ (iOS cần tương tác user trước khi play video)

### Yêu cầu bắt buộc
- **HTTPS** → GitHub Pages đã tự cung cấp ✅
- **User gesture** trước khi gọi `getUserMedia()` → nút bấm ✅
- Camera + microphone permission từ user

---

## Tính năng cần implement

### Core
- [x] Bật/tắt camera local
- [x] Hiển thị video local (muted, tránh echo)
- [x] Sinh Peer ID ngẫu nhiên từ PeerJS server
- [x] Copy Peer ID ra clipboard
- [x] Nhập Peer ID của đối phương → gọi
- [x] Nghe cuộc gọi đến tự động (auto-answer)
- [x] Hiển thị video remote
- [x] Cúp máy
- [x] Trạng thái kết nối (idle / connecting / connected / error)

### UX
- [x] Responsive layout (mobile + desktop)
- [x] Hiển thị rõ ID của mình để chia sẻ
- [x] Báo lỗi thân thiện khi camera bị từ chối
- [x] Indicator đang kết nối

### Nice-to-have (không bắt buộc)
- [ ] Mute mic
- [ ] Tắt camera trong khi đang call
- [ ] Chat text qua DataChannel
- [ ] Chất lượng video selector (HD/SD)

---

## Chi tiết implement

### HTML structure

```html
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>P2P Video Call</title>
</head>
<body>
  <!-- Video grid: local | remote -->
  <!-- Controls: start camera, peer ID input, call button, hang up -->
  <!-- Status bar: trạng thái kết nối -->
  <!-- My ID box: hiển thị + copy button -->
</body>
</html>
```

### JavaScript flow

```
1. User nhấn "Bật camera"
   → navigator.mediaDevices.getUserMedia({ video: true, audio: true })
   → localStream → hiển thị vào <video id="local-video" muted>
   → khởi tạo new Peer()

2. Peer 'open' event
   → nhận peer ID từ PeerJS server
   → hiển thị ID, enable nút "Gọi"

3a. Người gọi (Caller):
   → nhập ID đối phương → nhấn "Gọi"
   → peer.call(remoteId, localStream)
   → call.on('stream') → gán vào <video id="remote-video">

3b. Người nghe (Callee):
   → peer.on('call') → incomingCall.answer(localStream)
   → call.on('stream') → gán vào <video id="remote-video">

4. Cúp máy:
   → call.close()
   → reset remote video
```

### PeerJS config

```javascript
// Dùng server mặc định của PeerJS (miễn phí, HTTPS)
const peer = new Peer(); 
// Không cần config thêm gì — tự kết nối 0.peerjs.com:443

// Nếu muốn custom ID:
const peer = new Peer('my-custom-id');
```

### Video element setup

```html
<!-- Local: PHẢI có muted để tránh feedback âm thanh -->
<video id="local-video" autoplay muted playsinline></video>

<!-- Remote: KHÔNG muted -->
<video id="remote-video" autoplay playsinline></video>
```

> `playsinline` bắt buộc trên iOS Safari, nếu thiếu video không play được.

### Error handling cần cover

| Lỗi | Nguyên nhân | Xử lý |
|-----|-------------|-------|
| `NotAllowedError` | User từ chối camera/mic | Hiện hướng dẫn bật lại permission |
| `NotFoundError` | Không có camera/mic | Thông báo thiết bị không tìm thấy |
| `peer.on('error')` type `peer-unavailable` | Sai Peer ID | Thông báo "ID không tồn tại" |
| `peer.on('error')` type `network` | Mất kết nối internet | Thông báo và offer retry |
| `call.on('close')` | Đối phương cúp | Reset UI về trạng thái idle |

---

## CSS yêu cầu

```css
/* Video grid layout */
.video-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

/* Mobile: stack dọc */
@media (max-width: 600px) {
  .video-grid {
    grid-template-columns: 1fr;
  }
}

/* Video box: giữ tỉ lệ 16:9, background đen */
.video-box {
  background: #111;
  aspect-ratio: 16 / 9;
  border-radius: 12px;
  overflow: hidden;
  position: relative;
}

.video-box video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

---

## GitHub Pages deployment

### Setup (làm 1 lần)

```bash
# 1. Tạo repo mới trên GitHub (public hoặc private đều được)
# 2. Push code lên branch main
git init
git add .
git commit -m "init: WebRTC P2P app"
git remote add origin https://github.com/<username>/<repo>.git
git push -u origin main

# 3. Vào repo Settings → Pages → Source: "Deploy from branch"
# 4. Chọn branch: main, folder: / (root)
# 5. Save → GitHub tự build và deploy
```

### URL sau khi deploy
```
https://<username>.github.io/<repo>/
```

### Thời gian deploy: ~30-60 giây sau mỗi push

---

## Kiểm tra blockage trên GitHub Pages

### ✅ KHÔNG bị block

| Tính năng | Lý do |
|-----------|-------|
| `getUserMedia()` (camera/mic) | GitHub Pages là HTTPS → secure context ✅ |
| WebRTC `RTCPeerConnection` | Không cần HTTPS đặc biệt, chỉ cần browser hiện đại ✅ |
| PeerJS WebSocket signaling | Kết nối đến `wss://0.peerjs.com:443` (WSS = WebSocket Secure) ✅ |
| CDN từ `unpkg.com` | Không bị CORS block khi load script ✅ |
| P2P media stream | Trực tiếp giữa 2 browser, không qua GitHub server ✅ |

### ⚠️ CẦN LƯU Ý

| Vấn đề | Chi tiết | Giải pháp |
|---------|---------|-----------|
| **iOS Safari** | Cần `playsinline` trên `<video>`, cần user tap trước khi play | Đã cover trong code |
| **Camera permission** | User phải grant lần đầu | Hiện hướng dẫn rõ ràng trong UI |
| **NAT traversal** | Nếu 2 máy sau double-NAT nghiêm (hiếm) thì P2P fail | PeerJS free không có TURN → kết nối fail thay vì relay qua server lạ |
| **PeerJS free server** | Uptime không 100%, rate limit không rõ | Đủ dùng cho demo/nội bộ; production nên tự host PeerServer |
| **Mixed content** | Không được load script từ `http://` khi trang là `https://` | Dùng CDN HTTPS (unpkg.com đã HTTPS) ✅ |

### ❌ Những thứ GitHub Pages KHÔNG hỗ trợ (nhưng app này không cần)

- Server-side code (Node.js, Python, v.v.) → không cần
- WebSocket server riêng → không cần (dùng PeerJS cloud)
- Database → không cần
- File upload/storage → không cần

---

## README.md cho repo

```markdown
# P2P Video Call

Video call trực tiếp giữa 2 trình duyệt, không qua server trung gian.

## Cách dùng

1. Mở app trên 2 tab hoặc 2 máy
2. Cả 2 nhấn **"Bật camera"** và cho phép quyền truy cập
3. Máy A copy ID → gửi cho máy B (qua Zalo, chat, bất cứ đâu)
4. Máy B paste ID → nhấn **"Gọi"**
5. Video kết nối tự động

## Kỹ thuật

- Thuần HTML/CSS/JS, không có server
- WebRTC cho P2P media stream
- PeerJS cho signaling
- Deploy trên GitHub Pages (HTTPS)

## Giới hạn

- Cần internet để handshake lần đầu (dù cùng LAN)
- Không có TURN server → một số mạng có firewall nghiêm có thể không kết nối được
- PeerJS free server: dùng cho demo, không đảm bảo uptime
```

---

## Lệnh test local

```bash
# Python 3
python3 -m http.server 8080
# Mở: http://localhost:8080
# localhost được coi là secure context → getUserMedia hoạt động bình thường

# Node.js (nếu có)
npx serve .
```

> **Không mở file trực tiếp bằng `file:///`** — một số browser block `getUserMedia` với file:// protocol.

---

## Tóm tắt cho Claude Code

1. Tạo **1 file duy nhất** `index.html` chứa toàn bộ HTML + CSS + JS
2. Load PeerJS từ CDN: `https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js`
3. Implement đúng flow: camera → Peer init → show ID → call/answer → stream
4. Video local phải `muted`, cả 2 phải có `playsinline`
5. Handle đủ error cases (camera denied, wrong peer ID, connection drop)
6. CSS responsive: 2 cột desktop, 1 cột mobile
7. Push lên GitHub → enable Pages → done, không cần config thêm gì