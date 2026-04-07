# Adeleke-fees-payment

require('dotenv').config();
const express = require('express');
const axios = require('axios');
const bodyParser = require('body-parser');

const app = express();
app.use(bodyParser.json());
app.use(express.static('public')); // serve frontend

const PAYSTACK_SECRET = process.env.PAYSTACK_SECRET_KEY;
if (!PAYSTACK_SECRET) console.error('Set PAYSTACK_SECRET_KEY in .env');

app.post('/api/initiate-payment', async (req, res) => {
  // expected body: { email, amount_kobo, studentId, description }
  const { email, amount_kobo, studentId, description } = req.body;
  if (!email || !amount_kobo || !studentId) {
    return res.status(400).json({ error: 'Missing parameters' });
  }

  try {
    const initRes = await axios.post(
      'https://api.paystack.co/transaction/initialize',
      {
        email,
        amount: amount_kobo,
        metadata: { studentId, description },
      },
      { headers: { Authorization: `Bearer ${PAYSTACK_SECRET}` } }
    );
    return res.json({ authorization_url: initRes.data.data.authorization_url, reference: initRes.data.data.reference });
  } catch (err) {
    console.error(err.response?.data || err.message);
    return res.status(500).json({ error: 'Failed to initialize payment' });
  }
});

// webhook endpoint to confirm transaction (register this URL in Paystack dashboard)
app.post('/webhook/paystack', (req, res) => {
  // Validate via Paystack signature header in production: X-Paystack-Signature
  const event = req.body;
  // Example: verify signature using process.env.PAYSTACK_SECRET_KEY and raw body (see docs)
  // For brevity: assume valid. IMPORTANT: implement signature verification in production.

  if (event.event === 'charge.success') {
    const trx = event.data;
    const reference = trx.reference;
    const amount = trx.amount;
    const studentId = trx.metadata?.studentId;
    // TODO: update your DB: mark fees paid, store transaction
    console.log(`Payment success for student ${studentId}, ref ${reference}, amount ${amount}`);
  }
  res.status(200).send('ok');
});
VV
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server listening on ${PORT}`));
