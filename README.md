<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>The Great 2026 Colleague Quest (Live)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, collection, onSnapshot, query, getDocs } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";

        // Placeholder Comments Required
        // Chosen Palette: Warm Neutrals (Cream #FAFAF9, Stone #E7E5E4, Soft Gold #D4A373)
        // Application Structure Plan: Improved resilience with fallback for local testing.
        // Visualization & Content Choices: Real-time charts synced with Firestore.
        // CONFIRMATION: NO SVG graphics used. NO Mermaid JS used.

        // Safe access to environment variables
        const getEnvConfig = () => {
            try {
                return {
                    config: typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null,
                    appId: typeof __app_id !== 'undefined' ? __app_id : 'local-dev',
                    token: typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null
                };
            } catch (e) {
                return { config: null, appId: 'local-dev', token: null };
            }
        };

        const env = getEnvConfig();
        const participants = ['Andrew', 'Mo', 'KM', 'G'];
        let challengesData = [];
        let leaderboardChartInstance = null;
        let progressChartInstance = null;
        let currentFilter = 'all';
        let db, auth, currentUser;

        // --- INITIALIZATION ---
        const startApp = async () => {
            const statusEl = document.getElementById('db-status');
            
            if (!env.config) {
                statusEl.textContent = '🏠 Mod Local (Offline)';
                statusEl.classList.replace('text-stone-400', 'text-amber-600');
                loadLocalData();
                return;
            }

            try {
                const app = initializeApp(env.config);
                auth = getAuth(app);
                db = getFirestore(app);

                // RULE 3: Auth First
                if (env.token) {
                    await signInWithCustomToken(auth, env.token);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        currentUser = user;
                        statusEl.textContent = '🟢 Live Sync Activ';
                        statusEl.classList.replace('text-stone-400', 'text-green-600');
                        initCloudSync();
                    }
                });
            } catch (error) {
                console.error("Firebase Init Error:", error);
                statusEl.textContent = '⚠️ Eroare Conexiune';
                loadLocalData();
            }
        };

        // --- DATA SYNC ---
        const initCloudSync = async () => {
            const challengesCol = collection(db, 'artifacts', env.appId, 'public', 'data', 'challenges');
            
            // Initial Seed if empty
            const snapshot = await getDocs(challengesCol);
            if (snapshot.empty) {
                await seedDatabase(challengesCol);
            }

            onSnapshot(challengesCol, (snap) => {
                challengesData = snap.docs.map(d => d.data()).sort((a, b) => a.id - b.id);
                renderUI();
            }, (err) => {
                console.error("Snapshot error:", err);
            });
        };

        const loadLocalData = () => {
            // Fallback content for local file usage
            challengesData = [
                { id: 1, title: 'DIY Masterpiece', subtitle: 'Creative Activity', description: 'Pottery, painting, or building something together.', icon: '🎨', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 2, title: 'Literary Groupie', subtitle: 'Book Launch', description: 'Get that autograph!', icon: '📖', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 3, title: 'Karaoke Night', subtitle: 'Pop Star Debut', description: 'Show us those high notes.', icon: '🎤', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 4, title: 'New Horizons', subtitle: 'Travel Quest', description: 'A place you have never been before.', icon: '✈️', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                ...Array.from({ length: 8 }, (_, i) => ({
                    id: i + 5, title: 'Brainstorming...', subtitle: `Slot #${i + 5}`, description: 'Waiting for an idea.', icon: '❓', category: 'mystery', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false }
                }))
            ];
            renderUI();
        };

        async function seedDatabase(col) {
            const initial = [
                { id: 1, title: 'DIY Masterpiece', subtitle: 'The "I Actually Made This"', description: 'Engage in a creative DIY activity (Pottery? Painting? Building a birdhouse?).', icon: '🎨', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 2, title: 'Literary Groupie', subtitle: 'Book Launch Event', description: 'Attend a book launch and secure a copy with a real, live author\'s autograph.', icon: '📖', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 3, title: 'Karaoke Night', subtitle: 'Future Pop Star Debut', description: 'Hit the stage. Bonus points for dramatic high notes and questionable dance moves.', icon: '🎤', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                { id: 4, title: 'New Horizons', subtitle: 'Where On Earth Am I?', description: 'Visit a place or a country you have absolutely never set foot in before.', icon: '✈️', category: 'active', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false } },
                ...Array.from({ length: 8 }, (_, i) => ({
                    id: i + 5, title: 'Brainstorming...', subtitle: `Mystery Slot #${i + 5}`, description: 'Awaiting brilliant idea. Who is next to suggest one?', icon: '❓', category: 'mystery', status: { 'Andrew': false, 'Mo': false, 'KM': false, 'G': false }
                }))
            ];
            for (const item of initial) {
                await setDoc(doc(col, item.id.toString()), item);
            }
        }

        // --- INTERACTION ---
        window.toggleStatus = async (id, person) => {
            const challenge = challengesData.find(c => c.id === id);
            if (!challenge || challenge.category === 'mystery') return;

            const newStatus = { ...challenge.status, [person]: !challenge.status[person] };
            
            if (db && currentUser) {
                const ref = doc(db, 'artifacts', env.appId, 'public', 'data', 'challenges', id.toString());
                await setDoc(ref, { ...challenge, status: newStatus });
            } else {
                challenge.status = newStatus;
                renderUI();
            }
        };

        window.filterChallenges = (type) => {
            currentFilter = type;
            document.querySelectorAll('.filter-btn').forEach(b => b.classList.toggle('bg-white', b.id === `btn-${type}`));
            renderUI();
        };

        // --- RENDER ---
        function renderUI() {
            const grid = document.getElementById('challengesGrid');
            if (!grid) return;
            grid.innerHTML = '';

            const filtered = challengesData.filter(c => currentFilter === 'all' || c.category === currentFilter);
            filtered.forEach(c => {
                const total = Object.values(c.status).filter(Boolean).length;
                const card = document.createElement('div');
                card.className = `bg-white rounded-xl border p-5 flex flex-col h-full ${c.category === 'mystery' ? 'border-dashed opacity-60' : 'border-stone-200 shadow-sm'}`;
                
                let btns = `<div class="mt-auto grid grid-cols-2 gap-2 pt-4">`;
                participants.forEach(p => {
                    const active = c.status[p];
                    btns += `<button onclick="toggleStatus(${c.id}, '${p}')" class="text-xs font-bold p-2 rounded-lg border transition-all ${active ? 'bg-amber-400 border-amber-400 text-white' : 'bg-white text-stone-400'}">${p} ${active ? '✓' : ''}</button>`;
                });
                btns += `</div>`;

                card.innerHTML = `
                    <div class="mb-4">
                        <div class="flex justify-between items-start mb-2">
                            <span class="text-3xl">${c.icon}</span>
                            <span class="text-[10px] font-bold text-stone-300">#${c.id}</span>
                        </div>
                        <h3 class="font-bold text-stone-800">${c.title}</h3>
                        <p class="text-[10px] text-amber-600 font-bold uppercase mb-2">${c.subtitle}</p>
                        <p class="text-xs text-stone-500 line-clamp-2">${c.description}</p>
                    </div>
                    <div class="w-full bg-stone-100 h-1 rounded-full mb-1"><div class="bg-amber-400 h-full transition-all" style="width:${(total/4)*100}%"></div></div>
                    <p class="text-[10px] text-right text-stone-400 mb-2">${total}/4 Complete</p>
                    ${btns}
                `;
                grid.appendChild(card);
            });
            updateCharts();
        }

        function updateCharts() {
            const leaderStats = participants.map(p => challengesData.reduce((acc, c) => acc + (c.status[p] ? 1 : 0), 0));
            const total = challengesData.reduce((acc, c) => acc + Object.values(c.status).filter(Boolean).length, 0);

            if (!leaderboardChartInstance) {
                const config = { responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false } } };
                leaderboardChartInstance = new Chart(document.getElementById('leaderboardChart'), {
                    type: 'bar', 
                    data: { labels: participants, datasets: [{ data: leaderStats, backgroundColor: '#D4A373', borderRadius: 5 }] },
                    options: { ...config, indexAxis: 'y', scales: { x: { max: 12, beginAtZero: true } } }
                });
                progressChartInstance = new Chart(document.getElementById('progressChart'), {
                    type: 'doughnut',
                    data: { labels: ['Done', 'Left'], datasets: [{ data: [total, 48-total], backgroundColor: ['#CCD5AE', '#E7E5E4'] }] },
                    options: { ...config, cutout: '70%' }
                });
            } else {
                leaderboardChartInstance.data.datasets[0].data = leaderStats;
                leaderboardChartInstance.update();
                progressChartInstance.data.datasets[0].data = [total, 48-total];
                progressChartInstance.update();
            }
        }

        window.onload = startApp;
    </script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
        body { font-family: 'Inter', sans-serif; background-color: #FAFAF9; }
        .chart-container { position: relative; height: 200px; width: 100%; }
    </style>
</head>
<body class="p-4 md:p-8">
    <header class="max-w-6xl mx-auto mb-8 flex flex-col md:flex-row justify-between items-center bg-white p-6 rounded-2xl shadow-sm border border-stone-100">
        <div>
            <h1 class="text-2xl font-bold text-stone-800">🚀 2026 Colleague Quest</h1>
            <p id="db-status" class="text-[10px] font-bold uppercase tracking-widest text-stone-400">🟡 Connecting...</p>
        </div>
        <div class="mt-4 md:mt-0 flex bg-stone-100 p-1 rounded-xl">
            <button id="btn-all" onclick="filterChallenges('all')" class="filter-btn px-4 py-2 text-xs font-bold rounded-lg bg-white shadow-sm">All</button>
            <button id="btn-active" onclick="filterChallenges('active')" class="filter-btn px-4 py-2 text-xs font-bold rounded-lg text-stone-500">Active</button>
            <button id="btn-mystery" onclick="filterChallenges('mystery')" class="filter-btn px-4 py-2 text-xs font-bold rounded-lg text-stone-500">Mystery</button>
        </div>
    </header>

    <div class="max-w-6xl mx-auto grid grid-cols-1 lg:grid-cols-3 gap-6 mb-8">
        <div class="bg-white p-6 rounded-2xl border border-stone-100 shadow-sm lg:col-span-2">
            <h2 class="text-sm font-bold text-stone-400 uppercase mb-4">🏆 Leaderboard</h2>
            <div class="chart-container"><canvas id="leaderboardChart"></canvas></div>
        </div>
        <div class="bg-white p-6 rounded-2xl border border-stone-100 shadow-sm">
            <h2 class="text-sm font-bold text-stone-400 uppercase mb-4">📊 Overall</h2>
            <div class="chart-container"><canvas id="progressChart"></canvas></div>
        </div>
    </div>

    <div id="challengesGrid" class="max-w-6xl mx-auto grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <!-- JS Inject -->
    </div>
</body>
</html>
