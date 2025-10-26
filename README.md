# mi-webapp---

Estructura del proyecto

generador-ia/
├─ README.md
├─ package.json
├─ server/
│  ├─ index.js
│  ├─ webhook.js
│  └─ .env.example
└─ public/
   ├─ index.html
   ├─ main.js
   └─ style.css




OPENAI_API_KEY = (tu clave OpenAI)

STRIPE_SECRET_KEY = (tu clave Stripe)

STRIPE_PRICE_ID = (price ID de tu plan en Stripe para checkout)

STRIPE_WEBHOOK_SECRET = (para validar webhooks)

PORT = (opcional, por defecto 3000)



4. npm install


5. node server/index.js


6. Abrir http://localhost:3000 o el URL de Replit.



> Nota: este proyecto usa una implementación simple de control con acceso premiun




---

package.json

{
  "name": "generador-ia-micro-saas",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "stripe": "^12.10.0",
    "body-parser": "^1.20.2",
    "node-fetch": "^2.6.7",
    "cors": "^2.8.5"
  }
}


---

server/.env.example

PORT=3000
OPENAI_API_KEY=sk-REPLACE_ME
STRIPE_SECRET_KEY=sk_live_REPLACE_ME
STRIPE_PRICE_ID=price_XXXXXXXXXXXX
STRIPE_WEBHOOK_SECRET=whsec_XXXXXXXXXXXX


---

server/index.js

const express = require('express');
const bodyParser = require('body-parser');
const fetch = require('node-fetch');
const path = require('path');
const Stripe = require('stripe');
const cors = require('cors');

require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;
const OPENAI_KEY = process.env.OPENAI_API_KEY;
const stripe = Stripe(process.env.STRIPE_SECRET_KEY);

app.use(cors());
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, '..', 'public')));

// NOTA: en memoria — reemplaza por DB en producción
const activeSubscribers = new Map(); // email -> true

// Endpoint público para crear sesión de checkout
app.post('/api/create-checkout', async (req, res) => {
  const { email } = req.body;
  if (!email) return res.status(400).json({ error: 'Se requiere email' });

  try {
    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      mode: 'subscription',
      line_items: [{ price: process.env.STRIPE_PRICE_ID, quantity: 1 }],
      customer_email: email,
      success_url: ${req.headers.origin}/?success=1&email=${encodeURIComponent(email)},
      cancel_url: ${req.headers.origin}/?canceled=1,
    });

    res.json({ url: session.url });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Error creando sesión de pago' });
  }
});

// Webhook para marcar suscripciones activas (simplificado)
app.post('/webhook', bodyParser.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature'];
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  let event;

  try {
    event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
  } catch (err) {
    console.log('Webhook signature verification failed.', err.message);
    return res.status(400).send(Webhook Error: ${err.message});
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    // marcar email como suscrito (usar DB en la realidad)
    if (session.customer_email) {
      activeSubscribers.set(session.customer_email, true);
      console.log('Suscripción activada para', session.customer_email);
    }
  }

  res.json({ received: true });
});

// Endpoint que genera el texto usando OpenAI — requiere email suscrito
app.post('/api/generate', async (req, res) => {
  const { email, prompt } = req.body;
  if (!email || !prompt) return res.status(400).json({ error: 'Faltan parámetros' });

  // Verificar suscripción (simplificado)
  if (!activeSubscribers.get(email)) {
    return res.status(403).json({ error: 'No suscrito. Por favor paga para usar.' });
  }

  try {
    const resp = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': Bearer ${OPENAI_KEY}
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 800
      })
    });

    const data = await resp.json();
    const text = data?.choices?.[0]?.message?.content;
    res.json({ output: text });
  } catch (err) {
    console.error('OpenAI error', err);
    res.status(500).json({ error: 'Error generando texto' });
  }
});

app.listen(PORT, () => console.log(Server escuchando en ${PORT}));


---

public/index.html

<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Generador IA - Demo</title>
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <div class="container">
    <h1>Generador IA — Currículum / Carta</h1>

    <label>Email (para suscripción):</label>
    <input id="email" placeholder="tu@correo.com" />

    <button id="subscribeBtn">Pagar suscripción</button>

    <hr />

    <textarea id="prompt" placeholder="Describe lo que quieres generar..." rows="6"></textarea>
    <button id="generateBtn">Generar (solo suscriptores)</button>

    <pre id="output"></pre>
  </div>
  <script src="/main.js"></script>
</body>
</html>


---

public/style.css

body{font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial; padding:24px}
.container{max-width:720px;margin:0 auto}
input, textarea{width:100%;padding:8px;margin:8px 0}
button{padding:10px 14px;border-radius:8px;border:none;cursor:pointer}
pre{background:#f7f7f7;padding:12px;border-radius:8px}


---

public/main.js

const subscribeBtn = document.getElementById('subscribeBtn');
const generateBtn = document.getElementById('generateBtn');
const emailInput = document.getElementById('email');
const promptInput = document.getElementById('prompt');
const outputEl = document.getElementById('output');

subscribeBtn.onclick = async () => {
  const email = emailInput.value.trim();
  if (!email) return alert('Ingresa tu email');

  const res = await fetch('/api/create-checkout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email })
  });

  const data = await res.json();
  if (data.url) window.location = data.url;
  else alert('Error creando checkout');
};

generateBtn.onclick = async () => {
  const email = emailInput.value.trim();
  const prompt = promptInput.value.trim();
  if (!email || !prompt) return alert('Email y prompt son obligatorios');

  outputEl.textContent = 'Generando...';
  const res = await fetch('/api/generate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, prompt })
  });

  const data = await res.json();
  if (data.output) outputEl.textContent = data.output;
  else outputEl.textContent = 'Error: ' + (data.error || 'Respuesta inesperada');
};


---



---
