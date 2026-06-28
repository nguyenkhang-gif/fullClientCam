# P2P Video Call

Video call trực tiếp giữa 2 trình duyệt, không qua server trung gian.

## Cách dùng

1. Mở app trên 2 tab hoặc 2 máy
2. Cả 2 nhấn **"Bật camera"** và cho phép quyền truy cập
3. Máy A copy ID → gửi cho máy B (qua Zalo, chat, bất cứ đâu)
4. Máy B paste ID → nhấn **"Gọi"**
5. Video kết nối tự động

## Tính năng

- Bật/tắt camera + micro local
- Sinh & copy Peer ID
- Gọi / auto-answer cuộc gọi đến
- Cúp máy, mute mic, tắt camera trong khi gọi
- Chat text qua DataChannel
- Chọn chất lượng video (HD / SD)
- Responsive (desktop 2 cột, mobile 1 cột)

## Kỹ thuật

- Thuần HTML/CSS/JS, không có server, 1 file `index.html`
- WebRTC cho P2P media stream
- PeerJS (CDN unpkg) cho signaling
- Deploy trên GitHub Pages (HTTPS)

## Chạy local

```bash
python3 -m http.server 8080
# Mở: http://localhost:8080
```

> Đừng mở trực tiếp bằng `file:///` — browser sẽ block `getUserMedia`.

## Deploy GitHub Pages

```bash
git init
git add .
git commit -m "init: WebRTC P2P app"
git remote add origin https://github.com/<username>/<repo>.git
git push -u origin main
```

Sau đó: repo **Settings → Pages → Source: Deploy from branch → main / (root)**.
URL: `https://<username>.github.io/<repo>/`

## Giới hạn

- Cần internet để handshake lần đầu (dù cùng LAN)
- Không có TURN server → một số mạng firewall nghiêm có thể không kết nối được
- PeerJS free server: dùng cho demo, không đảm bảo uptime
