// index.js — Полный MVP (Express + Stripe + Frontend в одном файле)
// Установка: npm init -y && npm i express stripe body-parser
// Запуск локально: node index.js
// Деплой: загрузить на GitHub → Vercel → Node.js 18+

import express from "express";
import Stripe from "stripe";
import bodyParser from "body-parser";
import path from "path";
import { fileURLToPath } from "url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const app = express();
const PORT = process.env.PORT || 3000;

// ====== CONFIG ======
const STRIPE_SECRET = process.env.STRIPE_SECRET_KEY || "sk_test_xxx";
const STRIPE_PUBLIC = process.env.STRIPE_PUBLISHABLE_KEY || "pk_test_xxx";
const WEBHOOK_SECRET = process.env.STRIPE_WEBHOOK_SECRET || "";
const stripe = new Stripe(STRIPE_SECRET, { apiVersion: "2024-06-20" });

// ====== STORAGE ======
let tokens = []; // { idx, text, nick, url }

// ====== MIDDLEWARE ======
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

// ====== FRONTEND HTML ======
const html = (words) => `
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8" />
<title>Word Mayhem MVP</title>
<style>
  body { font-family: Arial, sans-serif; background: #f8faff; padding: 20px; }
  .word { display:inline-block; margin:3px; padding:4px 6px; border-radius:4px; background:#fff; cursor:pointer; }
  .word:hover { background: #d0e6ff; }
  .controls { margin-top: 10px; }
</style>
</head>
<body>
<h1>Word Mayhem</h1>
<div id="words">
${words.map(w => `<span class="word" title="${w.nick || ""}">${w.text}</span>`).join(" ")}
</div>
<div class="controls">
  <input id="nick" placeholder="Nick" />
  <input id="url" placeholder="Social link (optional)" />
  <input id="input" placeholder="Your words" />
  <button id="buy">Add to Story ($1 per token)</button>
</div>
<script src="https://js.stripe.com/v3/"></script>
<script>
const stripe = Stripe("${STRIPE_PUBLIC}");
document.getElementById("buy").onclick = async () => {
  const text = document.getElementById("input").value.trim();
  const nick = document.getElementById("nick").value.trim();
  const url = document.getElementById("url").value.trim();
  const tokens = text.split(" ").filter(t => t.length > 0 && t.length <= 16);
  if (tokens.length < 1 || tokens.length > 5) {
    alert("Enter 1-5 tokens (1-16 chars each)");
    return;
  }
  const res = await fetch("/create-checkout-session", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ tokens, nick, url })
  });
  const data = await res.json();
  stripe.redirectToCheckout({ sessionId: data.id });
};
</script>
</body>
</html>
`;

// ====== ROUTES ======
app.get("/", (req, res) => {
  res.send(html(tokens));
});

app.post("/create-checkout-session", async (req, res) => {
  const { tokens: toks, nick, url } = req.body;
  const amount = toks.length * 100; // $1 per token
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ["card"],
    line_items: [
      {
        price_data: {
          currency: "usd",
          product_data: { name: `${toks.length} word tokens` },
          unit_amount: 100
        },
        quantity: toks.length
      }
    ],
    mode: "payment",
    success_url: `${req.headers.origin}/?success=true`,
    cancel_url: `${req.headers.origin}/?canceled=true`,
    metadata: { toks: JSON.stringify(toks), nick, url }
  });
  res.json({ id: session.id });
});

app.post("/webhook", bodyParser.raw({ type: "application/json" }), (req, res) => {
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, req.headers["stripe-signature"], WEBHOOK_SECRET);
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }
  if (event.type === "checkout.session.completed") {
    const session = event.data.object;
    const toks = JSON.parse(session.metadata.toks);
    toks.forEach((t, i) => {
      tokens.push({
        idx: tokens.length + 1,
        text: t,
        nick: session.metadata.nick || "",
        url: session.metadata.url || ""
      });
    });
  }
  res.json({ received: true });
});

// ====== START ======
app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
