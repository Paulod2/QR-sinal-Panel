const path = require('path');
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });
const PORT = process.env.PORT || 3000;

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

let totalScans = 0;
const history = [];
const MAX_HISTORY = 100;

function getClientIp(req) {
  const forwarded = req.headers['x-forwarded-for'];
  if (forwarded) return forwarded.split(',')[0].trim();
  return (req.socket.remoteAddress || 'desconhecido').replace('::ffff:', '');
}

function isPrivateIp(ip) {
  return !ip || ip === 'desconhecido' || ip === '::1' ||
    ip.startsWith('127.') || ip.startsWith('10.') ||
    ip.startsWith('192.168.') || ip.startsWith('172.');
}

async function lookupGeo(ip) {
  if (isPrivateIp(ip)) return { city: 'Rede local', country: 'Ambiente de desenvolvimento' };
  try {
    const resp = await fetch(`http://ip-api.com/json/${ip}?fields=status,country,city`);
    const data = await resp.json();
    if (data.status === 'success') {
      return { city: data.city || 'Cidade desconhecida', country: data.country || 'País desconhecido' };
    }
  } catch (err) {
    console.error('Falha ao consultar geolocalização:', err.message);
  }
  return { city: 'Desconhecida', country: 'Desconhecido' };
}

function parseSimpleUserAgent(ua) {
  if (!ua) return 'Desconhecido';
  if (/android/i.test(ua)) return 'Android';
  if (/iphone|ipad|ios/i.test(ua)) return 'iOS';
  if (/windows/i.test(ua)) return 'Windows';
  if (/macintosh/i.test(ua)) return 'macOS';
  return ua.slice(0, 60);
}

app.get('/scan', async (req, res) => {
  const ip = getClientIp(req);
  const userAgent = req.headers['user-agent'] || 'Desconhecido';
  const timestamp = new Date().toISOString();
  const geo = await lookupGeo(ip);

  totalScans += 1;
  const record = { id: totalScans, timestamp, ip, city: geo.city, country: geo.country,
    device: parseSimpleUserAgent(userAgent), userAgent };

  history.unshift(record);
  if (history.length > MAX_HISTORY) history.pop();

  io.emit('QR_CODE_SCANNED', { total: totalScans, record });
  res.sendFile(path.join(__dirname, 'public', 'scan.html'));
});

app.get('/api/status', (req, res) => res.json({ total: totalScans, history }));

io.on('connection', (socket) => {
  socket.emit('init', { total: totalScans, history });
});

server.listen(PORT, () => {
  console.log(`Painel rodando em http://localhost:${PORT}`);
});
