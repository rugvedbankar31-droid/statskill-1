const http = require('http');
const fs = require('fs');
const path = require('path');

const ROOT = __dirname;
const DATA_FILE = path.join(ROOT, 'data.json');
const PORT = Number(process.env.PORT || 3000);

function readData() { return JSON.parse(fs.readFileSync(DATA_FILE, 'utf8')); }
function saveData(data) { fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2)); }
function send(res, status, body, type = 'application/json') {
  res.writeHead(status, { 'Content-Type': `${type}; charset=utf-8`, 'Cache-Control': 'no-store' });
  res.end(type === 'application/json' ? JSON.stringify(body) : body);
}
function getRequestBody(req) {
  return new Promise((resolve, reject) => {
    let raw = '';
    req.on('data', chunk => { raw += chunk; if (raw.length > 100000) req.destroy(); });
    req.on('end', () => { try { resolve(raw ? JSON.parse(raw) : {}); } catch { reject(new Error('Invalid JSON body')); } });
    req.on('error', reject);
  });
}
function moistureStatus(value) { return value < 30 ? 'Low — irrigation recommended' : value < 65 ? 'Ideal for tomatoes' : 'High — pause irrigation'; }
function sensorPayload(data) {
  return { ...data.sensors, moistureStatus: moistureStatus(data.sensors.soilMoisture), nutrientStatus: 'Good', waterDaysRemaining: Math.max(1, Math.floor(data.sensors.waterLevel / 22)) };
}
const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  const route = url.pathname;
  try {
    if (req.method === 'GET' && route === '/api/health') return send(res, 200, { ok: true, service: 'Agriminds API' });
    if (req.method === 'GET' && route === '/api/dashboard') { const data = readData(); return send(res, 200, { farm: data.farm, sensors: sensorPayload(data), pump: data.pump, alerts: data.alerts }); }
    if (req.method === 'GET' && route === '/api/sensors') return send(res, 200, sensorPayload(readData()));
    if (req.method === 'GET' && route === '/api/alerts') return send(res, 200, readData().alerts);
    if (req.method === 'GET' && route === '/api/irrigation') return send(res, 200, readData().pump);
    if (req.method === 'POST' && route === '/api/irrigation') {
      const update = await getRequestBody(req); const data = readData();
      if (typeof update.running !== 'boolean') return send(res, 400, { error: 'running must be true or false' });
      if (update.running && data.sensors.waterLevel <= 10) return send(res, 409, { error: 'Pump protected: water level is critically low.' });
      data.pump.running = update.running;
      data.pump.minutesRemaining = update.running ? (Number(update.minutesRemaining) || 18) : 0;
      saveData(data); return send(res, 200, data.pump);
    }
    if (req.method === 'POST' && route === '/api/sensors') {
      const update = await getRequestBody(req); const data = readData();
      const permitted = ['soilMoisture', 'nitrogen', 'phosphorus', 'potassium', 'temperature', 'humidity', 'waterLevel'];
      for (const key of permitted) if (typeof update[key] === 'number') data.sensors[key] = update[key];
      data.sensors.updatedAt = new Date().toISOString();
      if (data.sensors.waterLevel <= 10) data.pump.running = false;
      saveData(data); return send(res, 200, sensorPayload(data));
    }
    if (req.method === 'GET' && (route === '/' || route === '/index.html')) return send(res, 200, fs.readFileSync(path.join(ROOT, 'index.html'), 'utf8'), 'text/html');
    return send(res, 404, { error: 'Route not found' });
  } catch (error) { return send(res, 500, { error: error.message || 'Server error' }); }
});
server.listen(PORT, () => console.log(`Agriminds is running at http://localhost:${PORT}`));
