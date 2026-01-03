# Dynamic Consumer Registration for Kong Gateway

Consumers enable per-client quotas and authentication. Since Kong doesn't auto-create consumers from requests (to prevent abuse), this guide implements a **registration service** that clients POST to, which then securely creates the Consumer and API key in Kong.

## Step 1: Update Infrastructure

Update your `docker-compose.yml` to include the Node.js registration service:

```yaml
services:
  # ... existing kong + backend services ...

  register:
    image: node:20-alpine
    container_name: client-register
    working_dir: /app
    volumes:
      - ./register:/app
    ports:
      - "3000:3000"
    command: sh -c "npm install express axios && node server.js"
    networks:
      - kong-net
    depends_on:
      - kong

```

### Create `./register/package.json`

```json
{
  "name": "kong-register",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.19.2",
    "axios": "^1.7.2"
  }
}

```

---

## Step 2: Implementation (The Registration Logic)

Create `./register/server.js`. This service acts as a middleman between the public and the Kong Admin API.

```javascript
const express = require('express');
const axios = require('axios');
const app = express();
app.use(express.json());

const KONG_ADMIN = process.env.KONG_ADMIN || 'http://kong:8001';

app.post('/register', async (req, res) => {
  const { client_name, email } = req.body;

  // Basic Validation
  if (!client_name || !email || !email.includes('@')) {
    return res.status(400).json({ error: 'Missing/invalid client_name or email' });
  }

  try {
    // 1. Create Consumer in Kong
    const consumerRes = await axios.post(`${KONG_ADMIN}/consumers`, {
      username: `${client_name}-${Date.now()}`,
      custom_id: email
    });
    const consumer = consumerRes.data;

    // 2. Generate & assign API key
    const key = `key-${Date.now()}-${Math.random().toString(36).substr(-8)}`;
    await axios.post(`${KONG_ADMIN}/consumers/${consumer.id}/key-auth`, {
      key: key
    });

    console.log(`Created Consumer ${consumer.username} for ${email}`);

    res.json({
      success: true,
      consumer_id: consumer.id,
      username: consumer.username,
      apikey: key,
      proxy_url: 'http://localhost:8000/api',
      message: 'Use apikey in header: apikey: <key>'
    });
  } catch (error) {
    console.error('Registration failed:', error.response?.data);
    res.status(500).json({ error: 'Failed to register client' });
  }
});

app.get('/consumers', async (req, res) => {
  const { data } = await axios.get(`${KONG_ADMIN}/consumers`);
  res.json(data);
});

app.listen(3000, () => {
  console.log('Client registration service on http://localhost:3000');
});

```

---

## Step 3: Deployment & Verification

### Start the stack

```bash
docker compose down -v  # Optional: Clean slate
docker compose up -d

```

### Register your first client

```bash
curl -X POST http://localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "DemoApp",
    "email": "demo@example.com"
  }'

```

---

## Step 4: Testing Dynamic Access

Take the `apikey` from the previous response and use it to hit your Kong proxy:

```bash
curl -i http://localhost:8000/api/get?test=dynamic \
  -H "apikey: <YOUR_NEW_KEY>"

```

### Check Logs

* **Registration events:** `docker compose logs -f register`
* **Kong Proxy hits:** `docker compose logs -f kong`

---

## Step 5: Production Hardening (Local Demo)

To prevent spam, you can enhance `server.js` with a simple in-memory check or email uniqueness check:

```javascript
// Check for existing email (mock check)
const existing = await axios.get(`${KONG_ADMIN}/consumers?custom_id=${email}`);
if (existing.data.data.length > 0) {
  return res.status(409).json({ error: 'Email already registered' });
}

```

## Summary Flow

1. **Client** → POSTs to `/register` (port 3000).
2. **Register Service** → Validates and calls **Kong Admin API** (port 8001).
3. **Kong** → Creates Consumer + Key-Auth credential.
4. **Client** → Receives key and hits **Kong Proxy** (port 8000).

---

**Would you like me to help you write a script to load-test these dynamic consumers to see how the rate-limiting holds up?**
