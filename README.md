[index.html](https://github.com/user-attachments/files/23695797/index.html)
<!--
Single-file demo: index.html
Also included below (in this same document) is an example Node.js backend (server.js)
that simulates payment webhooks and uses socket.io to push real-time updates to the
laundry-operator dashboard.

Important notes (read before running):
- This is a demo. To accept real bank transfers and detect them automatically you must
  integrate with your bank's API or a payment provider that supports IBAN/payments and
  webhooks (not covered here).
- For production: use HTTPS, authentication, validate all incoming webhooks, store data
  in a secure database, and comply with local payment & data protection laws.
-->

<!doctype html>
<html lang="tr">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Halı Yıkama - Sipariş & Ödeme</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap" rel="stylesheet">
  <style>
    :root{
      --bg:#0f1724; --card:#0b1220; --accent:#16a34a; --muted:#94a3b8; --glass: rgba(255,255,255,0.03);
      --radius:14px; font-family:Inter, system-ui, Arial, sans-serif;
    }
    *{box-sizing:border-box}
    body{margin:0;min-height:100vh;background:linear-gradient(180deg,#071029 0%,#081225 100%);color:#e6eef8}
    .container{max-width:1100px;margin:36px auto;padding:28px}
    header{display:flex;align-items:center;justify-content:space-between;gap:12px}
    .brand{display:flex;gap:14px;align-items:center}
    .logo{width:56px;height:56px;border-radius:12px;background:linear-gradient(135deg,var(--accent),#059669);display:flex;align-items:center;justify-content:center;font-weight:700}
    h1{margin:0;font-size:20px}
    p.lead{margin:0;color:var(--muted)}

    .grid{display:grid;grid-template-columns:1fr 420px;gap:22px;margin-top:22px}
    .card{background:var(--card);border-radius:var(--radius);padding:18px;box-shadow:0 6px 30px rgba(2,6,23,0.6)}

    /* Price list */
    .price-list{display:grid;gap:10px}
    .price-item{display:flex;justify-content:space-between;padding:10px;border-radius:10px;background:var(--glass);align-items:center}
    .price-item b{font-weight:600}

    /* Order form */
    label{display:block;margin-top:10px;font-size:13px;color:var(--muted)}
    select,input[type=number],textarea{width:100%;padding:10px;margin-top:6px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit}
    .row{display:flex;gap:10px}
    button{background:var(--accent);border:0;padding:10px 14px;border-radius:10px;color:#071029;font-weight:600;cursor:pointer}
    .muted{color:var(--muted);font-size:13px}

    /* Right column */
    .right{position:sticky;top:24px}
    .iban-box{background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));padding:12px;border-radius:10px}
    .qr-wrap{display:flex;gap:12px;align-items:center}

    footer{margin-top:18px;color:var(--muted);font-size:13px;text-align:center}

    /* operator dashboard modal */
    .ops{display:flex;flex-direction:column;gap:8px}
    .order-card{background:rgba(20,28,38,0.6);padding:12px;border-radius:10px}
    .status-paid{color:#22c55e;font-weight:700}
    .status-pending{color:#f59e0b;font-weight:700}

    @media(max-width:980px){.grid{grid-template-columns:1fr}}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div class="brand">
        <div class="logo">HY</div>
        <div>
          <h1>Halı Yıkama - Online Sipariş</h1>
          <p class="lead">Kolay sipariş, IBAN ile ödeme, ve operatörde gerçek zamanlı bildirim.</p>
        </div>
      </div>
      <div class="muted">Canlı demo - Güvenlik & banka entegrasyonu ayrıca gereklidir.</div>
    </header>

    <div class="grid">
      <!-- LEFT: Price list + Order form -->
      <div class="card">
        <h3>Fiyat Listesi</h3>
        <div class="price-list" id="priceList"></div>

        <hr style="margin:14px 0;border:none;border-top:1px solid rgba(255,255,255,0.03)">

        <h3>Sipariş Oluştur</h3>
        <div>
          <label>Ürün</label>
          <select id="itemSelect">
            <!-- options injected by JS -->
          </select>

          <div class="row">
            <div style="flex:1">
              <label>Adet</label>
              <input id="qty" type="number" min="1" value="1">
            </div>
            <div style="width:160px">
              <label>İndirim (TL)</label>
              <input id="discount" type="number" min="0" value="0">
            </div>
          </div>

          <label>Not (tercihli)</label>
          <textarea id="note" rows="2" placeholder="Örn: lütfen sapları dikkatli paketleyin..."></textarea>

          <div style="display:flex;gap:10px;margin-top:12px;align-items:center">
            <button id="createOrder">Sipariş Oluştur & Ödeme Bilgisi Üret</button>
            <div class="muted" id="totalPreview">Toplam: 0 TL</div>
          </div>
        </div>

        <hr style="margin:14px 0;border:none;border-top:1px solid rgba(255,255,255,0.03)">

        <h3>Operatör Paneli (Canlı)</h3>
        <div class="ops" id="operatorList">
          <div class="muted">Henüz sipariş yok.</div>
        </div>
      </div>

      <!-- RIGHT: Payment box + QR -->
      <aside class="right">
        <div class="card">
          <h3>Ödeme Bilgisi</h3>
          <div class="iban-box">
            <div class="muted">IBAN</div>
            <div style="font-weight:700;font-size:16px;margin-top:6px" id="iban">TR00 0000 0000 0000 0000 0000 00</div>
            <div class="muted" style="margin-top:6px">Alıcı: Halı Yıkama A.Ş.</div>
            <hr style="margin:10px 0;border:none;border-top:1px solid rgba(255,255,255,0.02)">

            <div class="qr-wrap">
              <div id="qrcode"></div>
              <div>
                <div class="muted">Referans</div>
                <div id="payRef" style="font-weight:700;margin-top:6px">—</div>
                <div class="muted" style="margin-top:8px">Tutar</div>
                <div id="payAmount" style="font-weight:700;margin-top:6px">— TL</div>
              </div>
            </div>

            <div style="margin-top:12px;display:flex;gap:8px">
              <button id="copyIban">IBAN Kopyala</button>
              <button id="markPaid">(TEST) Ödeme Alındı</button>
            </div>

            <div class="muted" style="margin-top:10px;font-size:13px">Gerçek üretimde bankanızın IBAN/transfer webhooks veya bir ödeme sağlayıcısı kullanın.</div>
          </div>
        </div>

        <div class="card" style="margin-top:14px">
          <h3>Son İşlemler</h3>
          <div id="recent"></div>
        </div>
      </aside>
    </div>

    <footer>© Halı Yıkama - Demo. Gerçek ödemeler için banka entegrasyonu gereklidir.</footer>
  </div>

  <!-- QRCode.js from CDN -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
  <!-- socket.io client will be loaded from server in production; demo uses fallback polling -->

  <script>
    // --- Demo price data ---
    const PRICES = [
      {id:'standard', name:'Küçük Halı (0.5 - 2 m²)', price:65},
      {id:'large', name:'Büyük Halı (2 - 6 m²)', price:120},
      {id:'deep', name:'Derin Temizlik (her halı)', price:45},
      {id:'kilim', name:'Kilim / Şal', price:40}
    ];

    // UI refs
    const priceList = document.getElementById('priceList');
    const itemSelect = document.getElementById('itemSelect');
    const qtyInput = document.getElementById('qty');
    const discountInput = document.getElementById('discount');
    const totalPreview = document.getElementById('totalPreview');
    const createOrderBtn = document.getElementById('createOrder');
    const qrcodeWrap = document.getElementById('qrcode');
    const ibanEl = document.getElementById('iban');
    const payRef = document.getElementById('payRef');
    const payAmount = document.getElementById('payAmount');
    const operatorList = document.getElementById('operatorList');
    const recent = document.getElementById('recent');

    // demo IBAN - change to your IBAN
    const IBAN = 'TR33 0006 1005 1978 6457 8413 26';
    ibanEl.textContent = IBAN;

    // populate price list
    function renderPrices(){
      priceList.innerHTML=''; itemSelect.innerHTML='';
      PRICES.forEach(p=>{
        const row = document.createElement('div'); row.className='price-item';
        row.innerHTML = `<div>${p.name}</div><div><b>${p.price} TL</b></div>`;
        priceList.appendChild(row);
        const opt = document.createElement('option'); opt.value=p.id; opt.textContent=`${p.name} — ${p.price} TL`;
        itemSelect.appendChild(opt);
      });
    }

    // calculate total
    function calculate(){
      const item = PRICES.find(x=>x.id===itemSelect.value);
      const qty = Number(qtyInput.value)||1;
      const disc = Number(discountInput.value)||0;
      const total = Math.max(0, item.price*qty - disc);
      totalPreview.textContent = `Toplam: ${total} TL`;
      return total;
    }

    renderPrices(); calculate();
    itemSelect.onchange = qtyInput.oninput = discountInput.oninput = calculate;

    // utility: simple uuid
    function shortId(){return 'R'+Math.random().toString(36).slice(2,9).toUpperCase()}

    // create order -> show payment info & QR
    createOrderBtn.addEventListener('click', async ()=>{
      const id = shortId();
      const item = PRICES.find(x=>x.id===itemSelect.value);
      const qty = Number(qtyInput.value)||1;
      const discount = Number(discountInput.value)||0;
      const note = document.getElementById('note').value || '';
      const amount = Math.max(0, item.price*qty - discount);
      const order = {id, item:item.name, qty, amount, note, status:'pending', created: new Date().toISOString()};

      // save locally
      const list = JSON.parse(localStorage.getItem('orders')||'[]');
      list.unshift(order); localStorage.setItem('orders', JSON.stringify(list));

      // request server to create order record & broadcast to operator (if server present)
      try{
        await fetch('/api/create-order',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(order)});
      }catch(e){/*no server — fine for demo*/}

      // show payment box
      payRef.textContent = order.id;
      payAmount.textContent = `${order.amount} TL`;

      // create QR payload (simple text payload; real banks have specific QR standards per country)
      qrcodeWrap.innerHTML='';
      new QRCode(qrcodeWrap, {text:`IBAN:${IBAN}\nTutar:${order.amount} TL\nRef:${order.id}`, width:140, height:140});

      refreshRecent(); renderOperator();

      alert('Sipariş oluşturuldu. Müşteriye IBAN ve referans verin. Banka ödemeyi otomatik bildirim gönderirse sistem operatöre düşer. (Demo)');
    });

    // copy IBAN
    document.getElementById('copyIban').addEventListener('click',()=>{
      navigator.clipboard.writeText(IBAN).then(()=>alert('IBAN kopyalandı'));
    });

    // test mark paid (will call server webhook endpoint in demo)
    document.getElementById('markPaid').addEventListener('click', async ()=>{
      const list = JSON.parse(localStorage.getItem('orders')||'[]');
      if(!list.length) return alert('Önce bir sipariş oluşturun.');
      const order = list[0];
      // call server to simulate webhook
      try{
        await fetch('/api/simulate-payment',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({ref:order.id, amount:order.amount})});
      }catch(e){
        // fallback: mark locally
        order.status='paid'; list[0]=order; localStorage.setItem('orders',JSON.stringify(list));
        alert('(Demo) Ödeme kaydedildi (yerel). Operatör paneli güncellenecek.');
        renderOperator(); refreshRecent();
      }
    });

    // render recent orders (right panel)
    function refreshRecent(){
      const list = JSON.parse(localStorage.getItem('orders')||'[]');
      recent.innerHTML='';
      if(!list.length){recent.innerHTML='<div class="muted">Henüz işlem yok.</div>';return}
      list.slice(0,6).forEach(o=>{
        const d = document.createElement('div'); d.className='order-card';
        d.innerHTML = `<div><b>${o.item} x${o.qty}</b> — ${o.amount} TL</div><div class="muted">Ref: ${o.id}</div><div class="muted">${new Date(o.created).toLocaleString()}</div><div style="margin-top:6px">Durum: <span class="${o.status==='paid'?'status-paid':'status-pending'}">${o.status}</span></div>`;
        recent.appendChild(d);
      });
    }

    // operator panel render - tries to use socket.io if available; otherwise polls short local storage
    function renderOperator(orders){
      operatorList.innerHTML='';
      // try fetch from server
      fetch('/api/list-orders').then(r=>r.json()).then(data=>{
        if(!data || !data.length) operatorList.innerHTML='<div class="muted">Henüz sipariş yok.</div>';
        else data.forEach(o=>{
          const el = document.createElement('div'); el.className='order-card';
          el.innerHTML = `<div><b>${o.item} x${o.qty}</b></div><div class="muted">Ref: ${o.id}</div><div class="muted">${new Date(o.created).toLocaleString()}</div><div>Durum: <span class="${o.status==='paid'?'status-paid':'status-pending'}">${o.status}</span></div>`;
          operatorList.appendChild(el);
        });
      }).catch(e=>{
        // fallback to localStorage
        const list = JSON.parse(localStorage.getItem('orders')||'[]');
        if(!list.length) operatorList.innerHTML='<div class="muted">Henüz sipariş yok.</div>';
        else list.forEach(o=>{
          const el = document.createElement('div'); el.className='order-card';
          el.innerHTML = `<div><b>${o.item} x${o.qty}</b></div><div class="muted">Ref: ${o.id}</div><div class="muted">${new Date(o.created).toLocaleString()}</div><div>Durum: <span class="${o.status==='paid'?'status-paid':'status-pending'}">${o.status}</span></div>`;
          operatorList.appendChild(el);
        });
      });
    }

    // initial
    refreshRecent(); renderOperator();

    // quick polling so operator panel updates (demo only)
    setInterval(()=>{refreshRecent(); renderOperator();},5000);
  </script>

  <!--
  -----------------------------------------------------------------------------
  server.js (Node.js / Express demo)
  Save below as server.js (separate file) and run with: node server.js
  npm i express socket.io body-parser cors

  This simple demo stores orders in memory and broadcasts to connected operator clients.
  DO NOT use in production as-is. Use a persistent database and secure webhook verification.
  -----------------------------------------------------------------------------

  // ----- server.js START -----
  const express = require('express');
  const http = require('http');
  const { Server } = require('socket.io');
  const bodyParser = require('body-parser');
  const path = require('path');

  const app = express();
  const server = http.createServer(app);
  const io = new Server(server);

  app.use(bodyParser.json());
  app.use(require('cors')());

  // serve index.html (put the file in same folder)
  app.use('/', express.static(path.join(__dirname, '.')));

  // simple in-memory store (demo)
  const ORDERS = [];

  app.post('/api/create-order', (req, res)=>{
    const o = req.body; if(!o || !o.id) return res.status(400).send('bad');
    ORDERS.unshift({...o});
    io.emit('order-created', o);
    res.json({ok:true});
  });

  // endpoint to list orders
  app.get('/api/list-orders', (req,res)=>{
    res.json(ORDERS);
  });

  // simulate payment webhook (e.g., bank posts to you when transfer arrives)
  app.post('/api/simulate-payment', (req,res)=>{
    const {ref, amount} = req.body;
    const idx = ORDERS.findIndex(x=>x.id===ref);
    if(idx!==-1){
      ORDERS[idx].status='paid';
      io.emit('order-paid', ORDERS[idx]);
      return res.json({ok:true});
    }
    res.status(404).json({error:'not found'});
  });

  io.on('connection', socket=>{
    console.log('client connected');
  });

  const PORT = process.env.PORT || 3000;
  server.listen(PORT, ()=>console.log('Server running on',PORT));
  // ----- server.js END -----

  README / Quick run:
  1) Save index.html (this file) and server.js (above) in same folder.
  2) npm init -y
  3) npm i express socket.io body-parser cors
  4) node server.js
  5) Open http://localhost:3000 in both a "Müşteri" browser and a "Operatör" browser to see real-time updates.

  Production notes:
  - Use a real persistent DB (Postgres, MongoDB).
  - Use HTTPS and secure CORS rules.
  - For automatic detection of IBAN transfers: integrate with your bank's corporate API or use a payment provider (Garanti, Yapı Kredi, PayTR, Iyzico, etc.) that can send webhooks when a transfer with your reference arrives.
  - Always verify webhook signatures and amounts before marking orders paid.
  - Sanitize user inputs and implement rate limits.
  -->
</body>
</html>
