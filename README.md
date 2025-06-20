<!DOCTYPE html> <html lang="fr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Minage BTC - Simulateur avec Backend</title>
<style>
  body {
    background-color: #0d1117;
    color: #e6edf3;
    font-family: 'Inter', sans-serif;
    margin: 0; padding: 0;
    text-align: center;
  }
  header {
    background: #161b22;
    padding: 15px;
    position: sticky; top: 0; z-index: 100;
  }
  header button {
    margin: 0 10px;
    padding: 10px 20px;
    background: #f2a900;
    border: none; border-radius: 8px;
    cursor: pointer;
    font-weight: bold;
  }
  header button:hover {
    background: #ffd500;
  }
  .section {
    max-width: 900px;
    margin: 30px auto;
    background: #161b22;
    padding: 20px;
    border-radius: 15px;
    box-shadow: 0 0 15px #000;
    text-align: left;
  }
  .contract-item {
    background: #0d1117;
    padding: 12px;
    margin: 10px 0;
    border-radius: 10px;
  }
  .contract-item strong {
    color: #f2a900;
  }
  .btn-buy {
    background: #f2a900;
    border: none;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
    margin-top: 8px;
  }
  .btn-buy:hover {
    background: #ffd500;
  }
  #walletInfo {
    position: fixed;
    top: 80px; right: 20px;
    background: rgba(20,20,20,0.9);
    padding: 12px 18px;
    border-radius: 12px;
    font-weight: 600;
    box-shadow: 0 0 10px #fff1;
  }
  input[type=text] {
    width: 100%;
    padding: 6px;
    border-radius: 6px;
    border: none;
    margin-top: 6px;
    background: #22272e;
    color: #e6edf3;
  }
  #miningChart {
    max-width: 100%;
    height: 300px;
    margin-top: 20px;
  }
</style>
</head>
<body>

<header>
  <button onclick="showSection('contracts')">Contrats</button>
  <button onclick="showSection('active')">Contrats Actifs</button>
  <button onclick="showSection('pending')">En attente</button>
  <button onclick="showSection('profile')">Profil</button>
  <button onclick="showSection('withdraw')">Retrait</button>
</header>

<div id="contracts" class="section" style="display:none;">
  <h2>Contrats disponibles</h2>
  <div id="contractList"></div>
</div>

<div id="active" class="section" style="display:none;">
  <h2>Contrats Actifs</h2>
  <div id="activeContracts"></div>
  <canvas id="miningChart"></canvas>
</div>

<div id="pending" class="section" style="display:none;">
  <h2>Contrats en attente d'activation</h2>
  <div id="pendingContracts"></div>
</div>

<div id="profile" class="section" style="display:none;">
  <h2>Profil</h2>
  <p><strong>ID utilisateur:</strong> <span id="userId"></span></p>
  <p><strong>Pseudo:</strong><br>
    <input type="text" id="pseudo" oninput="saveData()" /></p>
  <p><strong>Ordinateur:</strong> <span id="computerName"></span></p>
</div>

<div id="withdraw" class="section" style="display:none;">
  <h2>Retrait</h2>
  <label>Adresse portefeuille BTC :</label><br>
  <input type="text" id="walletInput" placeholder="ex: 1BoatSLRH..." /><br><br>
  <button onclick="withdrawBTC()">Retirer</button>
</div>

<div id="walletInfo">Portefeuille: <span id="walletBalance">0.00000000</span> BTC</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
  const API_BASE = 'http://localhost:3000'; // Change selon backend

  // Contrats définis
  const contractStats = {
    0: { name: "Contrat Gratuit 24h", power: "10 TH/s", day: 0.0001, duration: 24*3600, price: 0 },
    1: { name: "Contrat 1", power: "50 TH/s", day: 0.0006, duration: 30*24*3600, price: 25 },
    2: { name: "Contrat 2", power: "100 TH/s", day: 0.0015, duration: 30*24*3600, price: 50 },
    3: { name: "Contrat 3", power: "200 TH/s", day: 0.0030, duration: 30*24*3600, price: 90 },
    4: { name: "Contrat 4", power: "500 TH/s", day: 0.0085, duration: 30*24*3600, price: 150 },
    5: { name: "Contrat 5", power: "1000 TH/s", day: 0.0180, duration: 30*24*3600, price: 280 },
  };

  let walletBalance = 0;
  let userId = '';
  let contractsActive = [];
  let pendingContracts = [];

  // Affichage sections
  function showSection(id) {
    ['contracts', 'active', 'pending', 'profile', 'withdraw'].forEach(s => {
      document.getElementById(s).style.display = s === id ? 'block' : 'none';
    });
  }

  // Génère ID utilisateur
  function generateUserId() {
    return 'ID-' + Math.random().toString(36).substr(2, 9);
  }

  // Récupère les contrats en attente depuis backend
  async function fetchPendingContracts() {
    try {
      const res = await fetch(`${API_BASE}/pending`);
      const data = await res.json();
      // Filtrer ceux du user
      pendingContracts = data.filter(p => p.userId === userId);
      updatePendingContracts();
    } catch(e) {
      console.error("Erreur fetch pending:", e);
    }
  }

  // Récupère les contrats actifs depuis backend
  async function fetchActiveContracts() {
    try {
      const res = await fetch(`${API_BASE}/active/${userId}`);
      const data = await res.json();
      contractsActive = data;
      updateActiveContracts();
    } catch(e) {
      console.error("Erreur fetch actifs:", e);
    }
  }

  // Affiche contrats disponibles
  function renderContracts() {
    const container = document.getElementById('contractList');
    container.innerHTML = '';
    for (const key in contractStats) {
      if (Object.hasOwnProperty.call(contractStats, key)) {
        const c = contractStats[key];
        const div = document.createElement('div');
        div.className = 'contract-item';
        div.innerHTML = `
          <strong>${c.name}</strong><br>
          Puissance: ${c.power}<br>
          Revenus: ${c.day} BTC/jour<br>
          Durée: ${c.duration/3600} heures<br>
          Prix: ${c.price === 0 ? 'Gratuit' : c.price + ' €'}<br>
          <button class="btn-buy" onclick="buyContract(${key})">${c.price === 0 ? 'Activer Gratuit' : 'Acheter'}</button>
        `;
        container.appendChild(div);
      }
    }
  }

  // Acheter un contrat (envoi au serveur)
  async function buyContract(level) {
    const c = contractStats[level];
    if (!c) return;

    if (c.price === 0) {
      // Contrat gratuit activé localement
      const now = Date.now();
      contractsActive.push({
        level,
        startTime: now,
        endTime: now + c.duration * 1000,
        minedBTC: 0
      });
      saveData();
      updateActiveContracts();
      alert("Contrat gratuit activé !");
      return;
    }

    // Ouvre PayPal (lien paypal) dans un nouvel onglet
    const paypalUrl = 'https://www.paypal.com/paypalme/cheatlegacy/' + c.price;
    window.open(paypalUrl, '_blank');

    // Ajout local en attente immédiatement
    const now = Date.now();
    pendingContracts.push({
      userId,
      level,
      timeAdded: now
    });
    saveData();
    updatePendingContracts();

    // Envoi au backend
    fetch(`${API_BASE}/buy`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({ userId, level })
    }).catch(e => console.error('Erreur envoi achat:', e));
  }

  // Affiche contrats actifs
  function updateActiveContracts() {
    const container = document.getElementById('activeContracts');
    container.innerHTML = '';

    const now = Date.now();
    // Nettoyer expirés
    contractsActive = contractsActive.filter(c => c.endTime > now);

    if(contractsActive.length === 0) {
      container.innerHTML = '<p>Aucun contrat actif.</p>';
    }

    contractsActive.forEach(c => {
      const contract = contractStats[c.level];
      const elapsedSec = (now - c.startTime) / 1000;
      const elapsedSecCapped = Math.min(elapsedSec, contract.duration);
      c.minedBTC = (elapsedSecCapped / 86400) * contract.day;

      const remainingSec = Math.max(0, (c.endTime - now) / 1000);
      const h = Math.floor(remainingSec / 3600);
      const m = Math.floor((remainingSec % 3600) / 60);

      const div = document.createElement('div');
      div.className = 'contract-item';
      div.innerHTML = `
        <strong>${contract.name}</strong><br>
        Puissance: ${contract.power}<br>
        Miné: ${c.minedBTC.toFixed(8)} BTC<br>
        Temps restant: ${h}h ${m}m
      `;
      container.appendChild(div);
    });
    saveData();
    updateWalletBalance();
    renderChart();
  }

  // Affiche contrats en attente
  function updatePendingContracts() {
    const container = document.getElementById('pendingContracts');
    container.innerHTML = '';

    if(pendingContracts.length === 0) {
      container.innerHTML = '<p>Aucun contrat en attente.</p>';
    }

    pendingContracts.forEach(p => {
      const contract = contractStats[p.level];
      const div = document.createElement('div');
      div.className = 'contract-item';
      const dateStr = new Date(p.timeAdded).toLocaleString();
      div.innerHTML = `
        <strong>${contract.name}</strong><br>
        Prix: ${contract.price} €<br>
        Commandé le: ${dateStr}
      `;
      container.appendChild(div);
    });
  }

  // Sauvegarde dans localStorage
  function saveData() {
    localStorage.setItem('userId', userId);
    localStorage.setItem('contractsActive', JSON.stringify(contractsActive));
    localStorage.setItem('pendingContracts', JSON.stringify(pendingContracts));
    localStorage.setItem('pseudo', document.getElementById('pseudo').value);
  }

  // Chargement depuis localStorage
  function loadData() {
    userId = localStorage.getItem('userId') || generateUserId();
    document.getElementById('userId').innerText = userId;

    const pseudo = localStorage.getItem('pseudo') || '';
    document.getElementById('pseudo').value = pseudo;

    contractsActive = JSON.parse(localStorage.getItem('contractsActive') || '[]');
    pendingContracts = JSON.parse(localStorage.getItem('pendingContracts') || '[]');

    updateActiveContracts();
    updatePendingContracts();
  }

  // Met à jour le solde (somme des contrats actifs)
  function updateWalletBalance() {
    walletBalance = contractsActive.reduce((acc, c) => acc + c.minedBTC, 0);
    document.getElementById('walletBalance').innerText = walletBalance.toFixed(8);
  }

  // Affiche le nom ordinateur (exemple simple)
  function showComputerName() {
    document.getElementById('computerName').innerText = navigator.platform || 'Inconnu';
  }

  // Retrait BTC (simple simulation)
  function withdrawBTC() {
    const walletAddr = document.getElementById('walletInput').value.trim();
    if(walletAddr.length < 10) {
      alert('Adresse portefeuille invalide.');
      return;
    }
    if(walletBalance < 0.0001) {
      alert('Solde insuffisant.');
      return;
    }
    alert(`Retrait de ${walletBalance.toFixed(8)} BTC vers ${walletAddr} demandé.`);
    walletBalance = 0;
    contractsActive = [];
    saveData();
    updateActiveContracts();
    document.getElementById('walletBalance').innerText = walletBalance.toFixed(8);
  }

  // Chart.js setup
  let miningChart = null;
  function renderChart() {
    const ctx = document.getElementById('miningChart').getContext('2d');
    const labels = contractsActive.map((c, i) => `Contrat ${c.level} #${i+1}`);
    const data = contractsActive.map(c => c.minedBTC);

    if(miningChart) miningChart.destroy();

    miningChart = new Chart(ctx, {
      type: 'bar',
      data: {
        labels: labels,
        datasets: [{
          label: 'BTC miné',
          data: data,
          backgroundColor: '#f2a900'
        }]
      },
      options: {
        scales: {
          y: { beginAtZero: true }
        }
      }
    });
  }

  // Initialisation
  window.onload = () => {
    loadData();
    showSection('contracts');
    renderContracts();
    showComputerName();

    // Synchronisation périodique des contrats
    setInterval(() => {
      fetchPendingContracts();
      fetchActiveContracts();
    }, 15000);
  }
</script>
</body>
</html>
