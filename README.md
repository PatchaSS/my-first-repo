<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>แดชบอร์ดสรุปผลความพึงพอใจ (ภาพรวมทั้งหมด)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-date-fns"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Sarabun', sans-serif;
            background-color: #f7fafc;
        }
        .kpi-card {
            background-color: white;
            border-radius: 1rem;
            padding: 1.5rem;
            box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
        }
        .loader {
            border: 5px solid #f3f3f3;
            border-top: 5px solid #F97316;
            border-radius: 50%;
            width: 50px;
            height: 50px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="bg-gray-100">

    <!-- Header -->
    <header class="bg-white shadow-md">
        <div class="container mx-auto max-w-7xl px-4 sm:px-6 lg:px-8 py-4 flex justify-between items-center">
            <h1 class="text-2xl font-bold text-gray-800">แดชบอร์ดความพึงพอใจ (ภาพรวมทั้งหมด)</h1>
            <div class="flex items-center space-x-2 text-sm text-gray-500">
                <span id="update-status-icon" class="w-3 h-3 bg-gray-400 rounded-full"></span>
                <span id="update-status-text">กำลังเชื่อมต่อ...</span>
            </div>
        </div>
    </header>

    <!-- Main Content -->
    <main class="container mx-auto max-w-7xl p-4 sm:p-6 lg:p-8">
        <!-- Loading State -->
        <div id="loading-state" class="text-center py-20">
            <div class="loader mx-auto"></div>
            <p class="mt-4 text-lg text-gray-600">กำลังดึงข้อมูลล่าสุดจาก Google Sheet...</p>
        </div>

        <!-- Dashboard Content -->
        <div id="dashboard-content" class="hidden">
            <!-- KPI Grid -->
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                <!-- Overall Score -->
                <div class="kpi-card col-span-1 md:col-span-2 lg:col-span-2 bg-orange-600 text-white flex flex-col justify-center items-center text-center">
                    <h2 class="text-xl font-semibold text-orange-100">ความพึงพอใจโดยรวม</h2>
                    <p id="kpi-overall" class="text-7xl font-bold my-2">0.0</p>
                    <p class="text-orange-200">จากผู้ประเมิน <span id="kpi-total-responses">0</span> คน</p>
                </div>
                <!-- Average Scores -->
                <div class="kpi-card text-center"><h2 class="text-lg text-gray-500">ด้านบุคลากร</h2><p id="kpi-personnel" class="text-5xl font-bold text-orange-500 mt-2">0.0</p></div>
                <div class="kpi-card text-center"><h2 class="text-lg text-gray-500">ด้านสถานที่</h2><p id="kpi-facility" class="text-5xl font-bold text-orange-500 mt-2">0.0</p></div>
                <div class="kpi-card text-center"><h2 class="text-lg text-gray-500">ด้านกระบวนการ</h2><p id="kpi-process" class="text-5xl font-bold text-orange-500 mt-2">0.0</p></div>
                <div class="kpi-card text-center"><h2 class="text-lg text-gray-500">ประเมินล่าสุดเมื่อ</h2><p id="kpi-latest-response" class="text-3xl font-bold text-orange-500 mt-2">-</p></div>
            </div>

            <!-- Charts Grid -->
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6 mt-6">
                <div class="kpi-card lg:col-span-2">
                    <h3 class="text-lg font-semibold text-gray-700 mb-4">แนวโน้มความพึงพอใจรายวัน</h3>
                    <canvas id="trendChart"></canvas>
                </div>
                <div class="kpi-card">
                    <h3 class="text-lg font-semibold text-gray-700 mb-4">การกระจายคะแนน</h3>
                    <canvas id="distributionChart"></canvas>
                </div>
            </div>

            <!-- Latest Comments -->
            <div class="kpi-card mt-6">
                 <h3 class="text-lg font-semibold text-gray-700 mb-4">ข้อเสนอแนะล่าสุด</h3>
                 <div id="latest-comments" class="space-y-4 max-h-96 overflow-y-auto pr-2">
                    <!-- Comments will be injected here -->
                 </div>
            </div>
        </div>
    </main>

    <script>
        // --- CONFIGURATION ---
        const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbwIM-eGP8JCfMy_hlvS3VvPhx7W7WGO_7vTWFQQm80VFUuvBFsAg3_gi3YZ4iLbckk0/exec';
        const REFRESH_INTERVAL = 30000; // 30 seconds

        // --- ELEMENTS ---
        const loadingState = document.getElementById('loading-state');
        const dashboardContent = document.getElementById('dashboard-content');
        const updateStatusIcon = document.getElementById('update-status-icon');
        const updateStatusText = document.getElementById('update-status-text');
        
        // KPI Elements
        const kpiOverall = document.getElementById('kpi-overall');
        const kpiTotalResponses = document.getElementById('kpi-total-responses');
        const kpiPersonnel = document.getElementById('kpi-personnel');
        const kpiFacility = document.getElementById('kpi-facility');
        const kpiProcess = document.getElementById('kpi-process');
        const kpiLatestResponse = document.getElementById('kpi-latest-response');
        const latestCommentsContainer = document.getElementById('latest-comments');

        // Chart instances
        let trendChart = null;
        let distributionChart = null;
        let allData = [];

        // --- FUNCTIONS ---

        async function fetchData() {
            console.log("Fetching data...");
            updateStatusText.textContent = 'กำลังอัปเดต...';
            updateStatusIcon.classList.add('animate-pulse', 'bg-yellow-400');
            updateStatusIcon.classList.remove('bg-gray-400', 'bg-green-500', 'bg-red-500');

            try {
                const response = await fetch(GOOGLE_SCRIPT_URL);
                if (!response.ok) throw new Error(`Network response was not ok: ${response.statusText}`);
                const data = await response.json();
                
                allData = data.map(row => {
                    // Convert score strings to numbers
                    for (const key in row) {
                        if (!isNaN(row[key]) && key !== 'timestamp' && row[key] !== '') {
                            row[key] = Number(row[key]);
                        }
                    }
                    return row;
                });

                loadingState.classList.add('hidden');
                dashboardContent.classList.remove('hidden');
                
                updateDashboard();

                updateStatusText.textContent = `อัปเดตล่าสุด: ${new Date().toLocaleTimeString('th-TH')}`;
                updateStatusIcon.classList.remove('animate-pulse', 'bg-yellow-400');
                updateStatusIcon.classList.add('bg-green-500');

            } catch (error) {
                console.error("Failed to fetch data:", error);
                updateStatusText.textContent = 'การเชื่อมต่อล้มเหลว';
                updateStatusIcon.classList.remove('animate-pulse', 'bg-yellow-400');
                updateStatusIcon.classList.add('bg-red-500');
            }
        }

        function updateDashboard() {
            const data = allData; // Use all data, no filtering

            if (data.length === 0) {
                kpiOverall.textContent = 'N/A';
                kpiTotalResponses.textContent = '0';
                return;
            }

            // Calculate KPIs
            const totalResponses = data.length;
            const overallSum = data.reduce((sum, row) => sum + (row.overall_satisfaction || 0), 0);
            const overallAvg = totalResponses > 0 ? (overallSum / totalResponses) : 0;

            const personnelAvg = calculateAverage(data, ['personnel_reception', 'personnel_nurse', 'personnel_doctor', 'personnel_manner']);
            const facilityAvg = calculateAverage(data, ['facility_room', 'facility_food', 'facility_equipment']);
            const processAvg = calculateAverage(data, ['process_speed', 'process_clarity']);

            // Update KPI Cards
            kpiOverall.textContent = overallAvg.toFixed(1);
            kpiTotalResponses.textContent = totalResponses;
            kpiPersonnel.textContent = personnelAvg.toFixed(1);
            kpiFacility.textContent = facilityAvg.toFixed(1);
            kpiProcess.textContent = processAvg.toFixed(1);
            
            const latestTimestamp = new Date(data[0].timestamp);
            kpiLatestResponse.textContent = latestTimestamp.toLocaleTimeString('th-TH');

            // Update Latest Comments
            latestCommentsContainer.innerHTML = data
                .filter(item => item.suggestions)
                .slice(0, 5)
                .map(item => `
                    <div class="p-3 bg-orange-50 rounded-lg border border-orange-100">
                        <p class="text-gray-700">"${item.suggestions}"</p>
                        <p class="text-xs text-right text-gray-500 mt-1">- ${item.department} | ${new Date(item.timestamp).toLocaleString('th-TH')}</p>
                    </div>
                `).join('') || '<p class="text-center text-gray-500">ยังไม่มีข้อเสนอแนะ</p>';

            // Update Charts
            updateDistributionChart(data);
            updateTrendChart(data);
        }

        function calculateAverage(data, keys) {
            let totalSum = 0;
            let totalCount = 0;
            data.forEach(row => {
                keys.forEach(key => {
                    if (row[key]) {
                        totalSum += row[key];
                        totalCount++;
                    }
                });
            });
            return totalCount > 0 ? (totalSum / totalCount) : 0;
        }

        function updateDistributionChart(data) {
            const counts = [0, 0, 0, 0, 0]; // for scores 1, 2, 3, 4, 5
            data.forEach(row => {
                const score = row.overall_satisfaction;
                if (score >= 1 && score <= 5) {
                    counts[score - 1]++;
                }
            });

            const chartData = {
                labels: ['1 คะแนน', '2 คะแนน', '3 คะแนน', '4 คะแนน', '5 คะแนน'],
                datasets: [{
                    label: 'จำนวนการประเมิน',
                    data: counts,
                    backgroundColor: [
                        '#EF4444', // red-500
                        '#F97316', // orange-500
                        '#FBBF24', // amber-400
                        '#A3E635', // lime-400
                        '#22C55E', // green-500
                    ],
                    borderColor: '#ffffff',
                    borderWidth: 2
                }]
            };

            if (distributionChart) {
                distributionChart.data = chartData;
                distributionChart.update();
            } else {
                distributionChart = new Chart(document.getElementById('distributionChart'), {
                    type: 'doughnut',
                    data: chartData,
                    options: {
                        responsive: true,
                        plugins: {
                            legend: { position: 'top' }
                        }
                    }
                });
            }
        }

        function updateTrendChart(data) {
            const dailyData = {};
            data.forEach(row => {
                const date = new Date(row.timestamp).toISOString().split('T')[0];
                if (!dailyData[date]) {
                    dailyData[date] = { sum: 0, count: 0 };
                }
                dailyData[date].sum += row.overall_satisfaction;
                dailyData[date].count++;
            });

            const sortedDates = Object.keys(dailyData).sort();
            const labels = sortedDates;
            const averages = sortedDates.map(date => dailyData[date].sum / dailyData[date].count);

            const chartData = {
                labels: labels,
                datasets: [{
                    label: 'คะแนนเฉลี่ยรายวัน',
                    data: averages,
                    fill: true,
                    borderColor: '#F97316',
                    backgroundColor: 'rgba(249, 115, 22, 0.1)',
                    tension: 0.3
                }]
            };
            
            if (trendChart) {
                trendChart.data = chartData;
                trendChart.update();
            } else {
                trendChart = new Chart(document.getElementById('trendChart'), {
                    type: 'line',
                    data: chartData,
                    options: {
                        scales: { y: { beginAtZero: true, max: 5 } },
                        plugins: { legend: { display: false } }
                    }
                });
            }
        }


        // --- INITIALIZATION ---
        document.addEventListener('DOMContentLoaded', () => {
            fetchData(); // Initial fetch
            setInterval(fetchData, REFRESH_INTERVAL); // Set auto-refresh
        });

    </script>
</body>
</html>


