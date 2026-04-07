#!/usr/bin/env bash
set -e

REPO_DIR="adeleke-fees"
ZIP_NAME="${REPO_DIR}.zip"

# Remove any existing dir/zip to avoid confusion
rm -rf "$REPO_DIR" "$ZIP_NAME"
mkdir -p "$REPO_DIR/db"
cd "$REPO_DIR"

# .gitignore
cat > .gitignore <<'EOF'
node_modules/
.env
.DS_Store
db/adeleke.db
EOF

# package.json
cat > package.json <<'EOF'
{
  "name": "adeleke-fees",
  "version": "1.0.0",
  "description": "Adeleke University fee payment demo (Paystack) — Railway-ready with a seeded student",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "init-db": "node db/init-db.js"
  },
  "dependencies": {
    "axios": "^1.5.0",
    "body-parser": "^1.20.2",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "sqlite3": "^5.1.6"
  }
}
EOF

# .env.example
cat > .env.example <<'EOF'
PORT=3000
PAYSTACK_SECRET_KEY=sk_test_xxx
PAYSTACK_PUBLIC_KEY=pk_test_xxx
EOF

# db/init-db.js
mkdir -p db
cat > db/init-db.js <<'EOF'
// Initializes SQLite DB and seeds one student record
const sqlite3 = require('sqlite3').verbose();
const path = require('path');
const dbPath = path.join(__dirname, 'adeleke.db');
const db = new sqlite3.Database(dbPath);

db.serialize(() => {
  // Create students table
  db.run(`CREATE TABLE IF NOT EXISTS students (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    matric_number TEXT UNIQUE,
    first_name TEXT,
    middle_name TEXT,
    last_name TEXT,
    department TEXT,
    email TEXT,
    outstanding_kobo INTEGER DEFAULT 0
  )`);

  // Create transactions table
  db.run(`CREATE TABLE IF NOT EXISTS transactions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    reference TEXT UNIQUE,
    student_matric TEXT,
    amount_kobo INTEGER,
    status TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )`);

  // Seed the student - Okolo Abigail Cletus, matric 24/1369, public health
  const stmt = db.prepare(`INSERT OR IGNORE INTO students
    (matric_number, first_name, middle_name, last_name, department, email, outstanding_kobo)
    VALUES (?, ?, ?, ?, ?, ?, ?)`);

  // Example outstanding fee: NGN 50,000 => 5,000,000 kobo
  stmt.run(
    '24/1369',
    'Abigail',
    'Cletus',
    'Okolo',
    'Public Health',
    'okolo.abigail@adelekeuniversity.edu',
    5000000
  );

  stmt.finalize();

  console.log('Database initialized and student seeded (Okolo Abigail Cletus, matric 24/1369).');
});

db.close();
EOF

# index.js
cat > index.js <<'EOF'
require('dotenv').config();
const express = require('express');
const axios = require('axios');
const bodyParser = require('body-parser');
const path = require('path');
const sqlite3 = require('sqlite3').verbose();
const fs = require('fs');

const app = express();
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

// Ensure DB exists (init if needed)
const dbPath = path.join(__dirname, 'db', 'adeleke.db');
if (!fs.existsSync(dbPath)) {
  console.log('DB not found. Initializing database...');
  require('./db/init-db');
}
const db = new sqlite3.Database(dbPath);

const PAYSTACK_SECRET = process.env.PAYSTACK_SECRET_KEY || '';
const PAYSTACK_PUBLIC = process.env.PAYSTACK_PUBLIC_KEY || '';

app.get('/health', (req, res) => res.json({ status: 'ok' }));

// Get student (by matric)
app.get('/api/student/:matric', (req, res) => {
  const { matric } = req.params;
  db.get('SELECT matric_number, first_name, middle_name, last_name, department, email, outstanding_kobo FROM students WHERE matric_number = ?', [matric], (err, row) => {
    if (err) return res.status(500).json({ error: 'DB error' });
    if (!row) return res.status(404).json({ error: 'Student not found' });
    return res.json({ student: row });
  });
});

// Initiate payment for a student
app.post('/api/initiate-payment', async (req, res) => {
  const { matric, email, amount_kobo, description } = req.body;
  if (!matric || !email || !amount_kobo) return res.status(400).json({ error: 'Missing parameters' });

  // Confirm student exists
  db.get('SELECT * FROM students WHERE matric_number = ?', [matric], async (err, student) => {
    if (err) return res.status(500).json({ error: 'DB error' });
    if (!student) return res.status(404).json({ error: 'Student not found' });

    try {
      const response = await axios.post('https://api.paystack.co/transaction/initialize', {
        email,
        amount: amount_kobo,
        metadata: { studentMatric: matric, description: description || 'School fees' }
      }, {
        headers: { Authorization: `Bearer ${PAYSTACK_SECRET}` }
      });

      const reference = response.data.data.reference;
      // Save a pending transaction record
      db.run(\`INSERT OR IGNORE INTO transactions (reference, student_matric, amount_kobo, status) VALUES (?, ?, ?, ?)\`,
        [reference, matric, amount_kobo, 'initialized']);

      return res.json({ authorization_url: response.data.data.authorization_url, reference });
    } catch (err) {
      console.error('Paystack init error:', err.response?.data || err.message);
      return res.status(500).json({ error: 'Failed to initialize payment' });
    }
  });
});

// Verify transaction server-side
app.get('/api/verify/:reference', async (req, res) => {
  const { reference } = req.params;
  if (!reference) return res.status(400).json({ error: 'Missing reference' });

  try {
    const response = await axios.get(\`https://api.paystack.co/transaction/verify/\${encodeURIComponent(reference)}\`, {
      headers: { Authorization: \`Bearer \${PAYSTACK_SECRET}\` }
    });
    const data = response.data.data;

    // If success, update transactions table and reduce student's outstanding balance
    if (data.status === 'success') {
      const matric = data.metadata?.studentMatric;
      const amount_kobo = data.amount;

      db.run('UPDATE transactions SET status = ? WHERE reference = ?', ['success', reference], (err) => {
        if (err) console.error('Failed to update transaction status', err);
      });

      if (matric) {
        db.run('UPDATE students SET outstanding_kobo = outstanding_kobo - ? WHERE matric_number = ?', [amount_kobo, matric], (err) => {
          if (err) console.error('Failed to update student outstanding balance', err);
        });
      }
    }

    return res.json(response.data);
  } catch (err) {
    console.error('Verify error:', err.response?.data || err.message);
    return res.status(500).json({ error: 'Failed to verify transaction' });
  }
});

// Webhook endpoint (register this in Paystack dashboard)
app.post('/webhook/paystack', (req, res) => {
  // IMPORTANT: Implement X-Paystack-Signature verification in production
  const event = req.body;
  if (event && event.event === 'charge.success') {
    const trx = event.data;
    const reference = trx.reference;
    const matric = trx.metadata?.studentMatric;
    const amount = trx.amount;

    // Update transaction to success if found
    db.run('UPDATE transactions SET status = ? WHERE reference = ?', ['success', reference], (err) => {
      if (err) console.error('Failed to update transaction', err);
    });

    // Adjust student outstanding balance
    if (matric) {
      db.run('UPDATE students SET outstanding_kobo = outstanding_kobo - ? WHERE matric_number = ?', [amount, matric], (err) => {
        if (err) console.error('Failed to update student outstanding balance', err);
        else console.log(\`Webhook processed: student \${matric} paid \${amount}\`);
      });
    }
  }
  res.sendStatus(200);
});

// Endpoint to get Paystack public key (safe to expose)
app.get('/api/public-key', (req, res) => {
  res.json({ publicKey: PAYSTACK_PUBLIC || '' });
});

// Serve frontend
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(\`Server listening on port \${PORT}\`));
EOF

# public files
mkdir -p public

# public/index.html
cat > public/index.html <<'EOF'
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Adeleke University — Pay Fees</title>
  <link rel="stylesheet" href="/style.css">
  <script src="https://js.paystack.co/v1/inline.js"></script>
</head>
<body>
  <div class="container">
    <h1>Adeleke University — Pay School Fees</h1>

    <div id="studentCard">
      <h2>Student Info</h2>
      <div id="studentInfo">Loading student data...</div>
    </div>

    <form id="payForm">
      <label>Amount to Pay (NGN)</label>
      <input id="amount" type="number" step="0.01" placeholder="e.g., 50000" required />
      <button type="submit">Pay Now</button>
    </form>

    <div id="message"></div>
  </div>

  <script src="/script.js"></script>
</body>
</html>
EOF

# public/style.css
cat > public/style.css <<'EOF'
body { font-family: Arial, sans-serif; background:#f6f9fc; }
.container { max-width:540px; margin:48px auto; padding:24px; background:#fff; border-radius:8px; box-shadow:0 2px 8px rgba(0,0,0,0.08); }
input { display:block; width:100%; margin:8px 0 16px; padding:8px; }
button { padding:10px 16px; background:#0070f3; color:white; border:none; border-radius:4px; cursor:pointer; }
#message { margin-top:16px; color:green; }
EOF

# public/script.js
cat > public/script.js <<'EOF'
(async function () {
  const STUDENT_MATRIC = '24/1369'; // pre-seeded student matric

  const studentInfoEl = document.getElementById('studentInfo');
  const messageEl = document.getElementById('message');

  function showMessage(msg, isError) {
    messageEl.style.color = isError ? 'red' : 'green';
    messageEl.textContent = msg;
  }

  // Load student data
  async function loadStudent() {
    const res = await fetch('/api/student/' + encodeURIComponent(STUDENT_MATRIC));
    if (!res.ok) {
      studentInfoEl.textContent = 'Student not found';
      return;
    }
    const data = await res.json();
    const s = data.student;
    const outstanding = (s.outstanding_kobo / 100).toFixed(2);
    studentInfoEl.innerHTML = `<strong>${s.first_name} ${s.middle_name} ${s.last_name}</strong><br>
      Matric: ${s.matric_number}<br>
      Department: ${s.department}<br>
      Email: ${s.email}<br>
      Outstanding: NGN ${outstanding}`;
  }

  await loadStudent();

  // Get public key from server
  async function getPublicKey() {
    const res = await fetch('/api/public-key');
    const j = await res.json();
    return j.publicKey;
  }

  document.getElementById('payForm').addEventListener('submit', async (e) => {
    e.preventDefault();
    const amountNgn = parseFloat(document.getElementById('amount').value);
    if (!amountNgn || isNaN(amountNgn) || amountNgn <= 0) return showMessage('Enter a valid amount', true);

    const amount_kobo = Math.round(amountNgn * 100);

    // Fetch student's email from DB (so payer uses student email)
    const stuRes = await fetch('/api/student/' + encodeURIComponent(STUDENT_MATRIC));
    const stuData = await stuRes.json();
    const email = stuData.student.email;

    // Initiate payment on server
    const init = await fetch('/api/initiate-payment', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({ matric: STUDENT_MATRIC, email, amount_kobo, description: 'School fees' })
    });
    const initData = await init.json();
    if (!init.ok) return showMessage('Init error: ' + (initData.error || 'Failed to initialize'), true);

    const publicKey = await getPublicKey();
    if (!publicKey) return showMessage('Public key not configured on server', true);

    // Open Paystack inline
    const handler = PaystackPop.setup({
      key: publicKey,
      email,
      amount: amount_kobo,
      reference: initData.reference,
      onClose: function(){ showMessage('Payment window closed', true); },
      callback: async function(response){
        showMessage('Payment complete! reference: ' + response.reference);
        // Optionally verify server-side
        const verify = await fetch('/api/verify/' + encodeURIComponent(response.reference));
        const verifyJson = await verify.json();
        console.log('Verify result:', verifyJson);
        setTimeout(loadStudent, 1000); // refresh student outstanding after a moment
      }
    });
    handler.openIframe();
  });

})();
EOF

# README.md
cat > README.md <<'EOF'
# Adeleke University — Fee Payment Demo

This is a minimal Node.js + Express app that demonstrates initializing Paystack payments and verifying transactions. It's prepared for deployment to Railway and pre-seeded with a student:

- Okolo Abigail Cletus — Matric: 24/1369 — Department: Public Health

Local setup
1. Copy .env.example to .env and set:
   PAYSTACK_SECRET_KEY=sk_test_xxx
   PAYSTACK_PUBLIC_KEY=pk_test_xxx
   PORT=3000

2. Install dependencies:
   npm install

3. Initialize the DB (creates db/adeleke.db and seeds the student):
   npm run init-db

4. Start the server:
   npm start

5. Open http://localhost:3000 — student data for 24/1369 is preloaded.

Deploy to Railway (quick)
1. Make a new GitHub repo, add these files, commit and push.
2. On Railway.app, create a new project → Deploy from GitHub → choose your repo.
3. Railway will detect Node. Set environment variables on Railway project:
   - PAYSTACK_SECRET_KEY = sk_test_xxx
   - PAYSTACK_PUBLIC_KEY = pk_test_xxx
4. (Optional) After deployment, trigger the DB init command in Railway console: npm run init-db
   - Alternatively, the first time the server runs it will auto-initialize if db is missing.
5. In Paystack Dashboard:
   - Add webhook URL: https://<your-railway-app-url>/webhook/paystack
   - Add allowed redirect/host URLs for inline if required by Paystack settings.

Security & production notes
- Do not commit secret keys to GitHub. Use Railway environment variables.
- Verify webhook signatures (X-Paystack-Signature) in production.
- Use a proper database for production if needed (Postgres), and add authentication for admins.
EOF

# go back up and install dependencies and run init-db
cd ..
echo "Created $REPO_DIR. Installing dependencies and creating database..."
cd "$REPO_DIR"
npm install --no-audit --no-fund >/dev/null 2>&1 || { echo "npm install failed. Please run 'npm install' in $REPO_DIR manually."; exit 1; }
npm run init-db >/dev/null 2>&1 || echo "Database init completed (or run 'npm run init-db' manually if needed)."

cd ..

# Zip the folder
zip -r "$ZIP_NAME" "$REPO_DIR" >/dev/null

echo "Done. Created $ZIP_NAME in $(pwd)"
echo "Next steps:"
echo "  - Unzip, inspect files, set .env values, and push to GitHub."
echo "  - Deploy to Railway and set PAYSTACK keys as environment variables there."
