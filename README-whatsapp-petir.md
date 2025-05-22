# 📦 WhatsApp Puppeteer Bot (Dockerized)

This project runs a WhatsApp automation bot using [whatsapp-web.js](https://github.com/pedroslopez/whatsapp-web.js) inside a Docker container on your VPS. It sends WhatsApp messages to a specified number, useful for alerts, notifications, or custom server events.

## ✅ Features

- Docker-based headless Chromium (no desktop needed)
- Secure QR login via WhatsApp Web
- Sends messages to external WhatsApp numbers
- Supports persistent login across container restarts
- Fully VPS-compatible and lightweight

## 🧱 Requirements

- Ubuntu VPS (x86_64)
- Docker installed
- Node.js project with:
  - `whatsapp-web.js`
  - `qrcode-terminal`

## 📁 Project Structure

```
whatsapp-petir/
├── Dockerfile
├── index.js
├── package.json
├── package-lock.json
└── .wwebjs_auth/ (auto-created for session persistence)
```

## ⚙️ Installation Steps

### 1. Clone or Create Directory

```bash
mkdir ~/whatsapp-petir && cd ~/whatsapp-petir
```

### 2. Create Files

#### `package.json`
```json
{
  "name": "whatsapp-petir",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "whatsapp-web.js": "^1.23.0",
    "qrcode-terminal": "^0.12.0"
  }
}
```

#### `index.js`
```js
const { Client, LocalAuth } = require('whatsapp-web.js');
const qrcode = require('qrcode-terminal');

console.log('📦 Starting WhatsApp bot...');

const client = new Client({
  authStrategy: new LocalAuth({ dataPath: '/root/.wwebjs_auth' }),
  puppeteer: {
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox'],
  },
});

client.on('qr', (qr) => {
  console.log('🔑 Scan this QR with your phone:');
  qrcode.generate(qr, { small: true });
});

client.on('auth_failure', (msg) => {
  console.error('❌ Authentication failure:', msg);
});

client.on('disconnected', (reason) => {
  console.error('❌ Client disconnected. Reason:', reason);
});

client.on('error', (err) => {
  console.error('❌ Client error:', err);
});

client.on('ready', async () => {
  console.log('🎉 Client is ready!');

  const rawNumber = '628XXXXXXXXX'; // Your recipient number
  const chatId = `${rawNumber}@c.us`;
  console.log(`🔍 Checking if ${chatId} is registered...`);

  const isRegistered = await client.isRegisteredUser(chatId);
  console.log(`✅ isRegisteredUser result: ${isRegistered}`);

  if (!isRegistered) {
    console.error('❌ Number is not registered on WhatsApp.');
    return;
  }

  const result = await client.sendMessage(chatId, 'Hello from Docker! ✅');
  console.log('📩 Message sent!');

  setTimeout(() => {
    console.log('⏹ Exiting...');
    process.exit(0);
  }, 10000);
});

client.on('message_ack', (msg, ack) => {
  console.log(`🟢 Message ${msg.id.id} delivery status: ${ack}`);
});

console.log('⚙️ Initializing WhatsApp Web session...');
client.initialize();
```

#### `Dockerfile`
```Dockerfile
FROM node:18-slim

RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    libatk-bridge2.0-0 \
    libnspr4 \
    libnss3 \
    libxss1 \
    libasound2 \
    libatk1.0-0 \
    libgtk-3-0 \
    libx11-xcb1 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libgbm1 \
    --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["node", "index.js"]
```

## 🛠️ Build & Run

### 1. Build the Docker image

```bash
docker build -t whatsapp-petir .
```

### 2. Run the bot and scan the QR

```bash
docker run -it --rm \
  -v $(pwd)/.wwebjs_auth:/root/.wwebjs_auth \
  --name whatsapp \
  whatsapp-petir
```

> ✅ The QR code will appear in your terminal. Scan it with your spare WhatsApp account.

## 🔁 Future Runs

```bash
docker run -it --rm \
  -v $(pwd)/.wwebjs_auth:/root/.wwebjs_auth \
  --name whatsapp \
  whatsapp-petir
```

## ✅ Output Example

```
📦 Starting WhatsApp bot...
⚙️ Initializing WhatsApp Web session...
🔑 Scan this QR with your phone:
🎉 Client is ready!
🔍 Checking if 628XXXXXXXXX@c.us is registered...
✅ isRegisteredUser result: true
📩 Message sent!
⏹ Exiting...
```

## 📌 Notes

- Ensure your recipient is using WhatsApp and not banned or inactive.
- This bot is **one-way** by default — it only sends messages.
- You can extend it to receive commands or respond dynamically.
- Chromium is run headless inside Docker.

## 📥 Future Ideas

- Run from `cron` or system events
- Integrate with alerting scripts
- Build an express API to trigger messages from anywhere

## 🧑‍💻 Author

Nazhif Setya — VPS Automation Enthusiast 🚀