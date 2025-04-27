// IndexedDB setup
let db;
const request = indexedDB.open('medicoes', 1);

request.onupgradeneeded = function(event) {
    db = event.target.result;
    const objectStore = db.createObjectStore('historico', { keyPath: 'id', autoIncrement: true });
};

request.onsuccess = function(event) {
    db = event.target.result;
    loadHistory();
};

request.onerror = function(event) {
    console.error('Erro no IndexedDB', event.target.errorCode);
};




// Variáveis principais
let connectedDevices = [];
let characteristics = [];
let sensorCount = 1;
let dataBySensor = [];
let collecting = false;
let clientName = "";
let equipmentName = "";
let imageBase64 = "";




// Login
function login() {
    const user = document.getElementById('username').value;
    const pass = document.getElementById('password').value;
    if (user === defaultUser && pass === defaultPass) {
        document.getElementById('loginPage').style.display = 'none';
        document.getElementById('mainPage').style.display = 'block';
        loadHistory();
        createCharts();
    } else {
        document.getElementById('loginError').textContent = "Usuário ou senha incorretos.";
    }
}


let accelerationChart, velocityChart, displacementChart, fftChartX, fftChartY, fftChartZ;

function createCharts() {
    const ctxAcc = document.getElementById('accelerationChart').getContext('2d');
    accelerationChart = new Chart(ctxAcc, {
        type: 'line',
        data: {
            labels: [],
            datasets: [
                { label: 'Acc X', data: [], borderColor: '#4CAF50', fill: false },
                { label: 'Acc Y', data: [], borderColor: '#2196F3', fill: false },
                { label: 'Acc Z', data: [], borderColor: '#FF9800', fill: false }
            ]
        },
        options: { responsive: true }
    });

    const ctxVel = document.getElementById('velocityChart').getContext('2d');
    velocityChart = new Chart(ctxVel, {
        type: 'line',
        data: {
            labels: [],
            datasets: [
                { label: 'Vel X', data: [], borderColor: '#4CAF50', fill: false },
                { label: 'Vel Y', data: [], borderColor: '#2196F3', fill: false },
                { label: 'Vel Z', data: [], borderColor: '#FF9800', fill: false }
            ]
        },
        options: { responsive: true }
    });

    const ctxDisp = document.getElementById('displacementChart').getContext('2d');
    displacementChart = new Chart(ctxDisp, {
        type: 'line',
        data: {
            labels: [],
            datasets: [
                { label: 'Disp X', data: [], borderColor: '#4CAF50', fill: false },
                { label: 'Disp Y', data: [], borderColor: '#2196F3', fill: false },
                { label: 'Disp Z', data: [], borderColor: '#FF9800', fill: false }
            ]
        },
        options: { responsive: true }
    });

    const ctxFftX = document.getElementById('fftChartX').getContext('2d');
    fftChartX = new Chart(ctxFftX, {
        type: 'line',
        data: { labels: [], datasets: [{ label: 'FFT X', data: [], borderColor: '#e91e63', fill: false }] },
        options: { responsive: true }
    });

    const ctxFftY = document.getElementById('fftChartY').getContext('2d');
    fftChartY = new Chart(ctxFftY, {
        type: 'line',
        data: { labels: [], datasets: [{ label: 'FFT Y', data: [], borderColor: '#9c27b0', fill: false }] },
        options: { responsive: true }
    });

    const ctxFftZ = document.getElementById('fftChartZ').getContext('2d');
    fftChartZ = new Chart(ctxFftZ, {
        type: 'line',
        data: { labels: [], datasets: [{ label: 'FFT Z', data: [], borderColor: '#3f51b5', fill: false }] },
        options: { responsive: true }
    });
}

// Limites de vibração por classe e zona (ISO 10816)
const vibration_limits = [
    [0.71, 1.8, 4.5, 4.5], // Classe I
    [1.12, 2.8, 7.1, 7.1], // Classe II
    [1.8, 4.5, 11.2, 11.2], // Classe III
    [2.8, 7.1, 18.0, 18.0]  // Classe IV
  ];
  


function logout() {
    document.getElementById('loginPage').style.display = 'flex';
    document.getElementById('mainPage').style.display = 'none';
}

// Conectar Sensor(es)
async function connectSensor() {
    sensorCount = parseInt(document.getElementById('sensorCount').value);
    connectedDevices = [];
    characteristics = [];
    dataBySensor = [];

    for (let i = 0; i < sensorCount; i++) {
        try {
            const device = await navigator.bluetooth.requestDevice({
                filters: [{ namePrefix: 'WT901' }],
                optionalServices: ['0000ffe5-0000-1000-8000-00805f9a34fb']
            });
            const server = await device.gatt.connect();
            const service = await server.getPrimaryService('0000ffe5-0000-1000-8000-00805f9a34fb');
            const characteristic = await service.getCharacteristic('0000ffe4-0000-1000-8000-00805f9a34fb');

            connectedDevices.push(device);
            characteristics.push(characteristic);
            dataBySensor.push({ accX: [], accY: [], accZ: [], velX: [], velY: [], velZ: [], dispX: [], dispY: [], dispZ: [] });

            alert(`Sensor ${i + 1} conectado: ${device.name}`);
        } catch (error) {
            alert("Erro ao conectar sensor " + (i + 1) + ": " + error);
        }
    }
}

// Iniciar Coleta
function startCollection() {
    if (characteristics.length === 0) {
        alert("Nenhum sensor conectado!");
        return;
    }

    clientName = document.getElementById('clientName').value;
    equipmentName = document.getElementById('equipmentName').value;
    const imgFile = document.getElementById('imageUpload').files[0];

    if (!clientName || !equipmentName) {
        alert("Preencha Cliente e Equipamento!");
        return;
    }

    if (imgFile) {
        const reader = new FileReader();
        reader.onload = function(e) {
            imageBase64 = e.target.result;
        };
        reader.readAsDataURL(imgFile);
    } else {
        imageBase64 = "";
    }

    dataBySensor.forEach(sensor => {
        sensor.accX = [];
        sensor.accY = [];
        sensor.accZ = [];
        sensor.velX = [];
        sensor.velY = [];
        sensor.velZ = [];
        sensor.dispX = [];
        sensor.dispY = [];
        sensor.dispZ = [];
    });

    collecting = true;

    characteristics.forEach((ch, index) => {
        ch.startNotifications().then(() => {
            ch.addEventListener('characteristicvaluechanged', (event) => handleSensorData(event, index));
        });
    });

    alert("Coleta iniciada!");
}

// Parar Coleta
function stopCollection() {
    if (collecting) {
        characteristics.forEach((ch) => {
            ch.stopNotifications();
        });
        collecting = false;

        saveMeasurement();
        loadHistory();

    }
}



// Processar Dados Recebidos
function handleSensorData(event, sensorIndex) {
    const data = new DataView(event.target.value.buffer);
    if (data.getUint8(0) !== 0x55 || data.getUint8(1) !== 0x61) {
        return;
    }

    const axRaw = data.getInt16(2, true);
    const ayRaw = data.getInt16(4, true);
    const azRaw = data.getInt16(6, true);

    const gxRaw = data.getInt16(8, true);
    const gyRaw = data.getInt16(10, true);
    const gzRaw = data.getInt16(12, true);

    const ax = axRaw / 32768 * 16;
    const ay = ayRaw / 32768 * 16;
    const az = azRaw / 32768 * 16;

    const gx = gxRaw / 32768 * 2000;
    const gy = gyRaw / 32768 * 2000;
    const gz = gzRaw / 32768 * 2000;

    dataBySensor[sensorIndex].accX.push(ax);
    dataBySensor[sensorIndex].accY.push(ay);
    dataBySensor[sensorIndex].accZ.push(az);

    dataBySensor[sensorIndex].velX.push(gx);
    dataBySensor[sensorIndex].velY.push(gy);
    dataBySensor[sensorIndex].velZ.push(gz);

    dataBySensor[sensorIndex].dispX.push(integrate(dataBySensor[sensorIndex].velX));
    dataBySensor[sensorIndex].dispY.push(integrate(dataBySensor[sensorIndex].velY));
    dataBySensor[sensorIndex].dispZ.push(integrate(dataBySensor[sensorIndex].velZ));

    plotAll(sensorIndex);
}

// Integração simples para deslocamento
function integrate(data) {
    let sum = 0;
    for (let i = 0; i < data.length; i++) {
        sum += data[i] * 0.02;
    }
    return sum;
}

// Plotar todos os gráficos
function plotAll(sensorIndex) {
    plotAcceleration(dataBySensor[sensorIndex]);
    plotVelocity(dataBySensor[sensorIndex]);
    plotDisplacement(dataBySensor[sensorIndex]);
    plotFFT(dataBySensor[sensorIndex]);
}

// Plotar Aceleração
function plotAcceleration(sensorData) {
    accelerationChart.data.labels = sensorData.accX.map((_, i) => i);
    accelerationChart.data.datasets[0].data = sensorData.accX;
    accelerationChart.data.datasets[1].data = sensorData.accY;
    accelerationChart.data.datasets[2].data = sensorData.accZ;
    accelerationChart.update();
}

// Plotar Velocidade
function plotVelocity(sensorData) {
    velocityChart.data.labels = sensorData.velX.map((_, i) => i);
    velocityChart.data.datasets[0].data = sensorData.velX;
    velocityChart.data.datasets[1].data = sensorData.velY;
    velocityChart.data.datasets[2].data = sensorData.velZ;
    velocityChart.update();
}

// Plotar Deslocamento
function plotDisplacement(sensorData) {
    displacementChart.data.labels = sensorData.dispX.map((_, i) => i);
    displacementChart.data.datasets[0].data = sensorData.dispX;
    displacementChart.data.datasets[1].data = sensorData.dispY;
    displacementChart.data.datasets[2].data = sensorData.dispZ;
    displacementChart.update();
}

// FFT simples
function fftSimple(data) {
    let n = data.length;
    let result = [];

    for (let i = 0; i < n; i++) {
        let re = 0, im = 0;
        for (let j = 0; j < n; j++) {
            let angle = (2 * Math.PI * i * j) / n;
            re += data[j] * Math.cos(angle);
            im -= data[j] * Math.sin(angle);
        }
        result.push(Math.sqrt(re * re + im * im));
    }
    return result;
}

// Plotar FFTs
function plotFFT(sensorData) {
    const fftX = fftSimple(sensorData.accX);
    const fftY = fftSimple(sensorData.accY);
    const fftZ = fftSimple(sensorData.accZ);

    fftChartX.data.labels = fftX.map((_, i) => i);
    fftChartX.data.datasets[0].data = fftX;
    fftChartX.update();

    fftChartY.data.labels = fftY.map((_, i) => i);
    fftChartY.data.datasets[0].data = fftY;
    fftChartY.update();

    fftChartZ.data.labels = fftZ.map((_, i) => i);
    fftChartZ.data.datasets[0].data = fftZ;
    fftChartZ.update();
}


// Diagnóstico automático atualizado
function generateDiagnosis(sensorData) {
    const maxAcc = Math.max(...sensorData.accX.map(Math.abs));
    const maxVel = Math.max(...sensorData.velX.map(Math.abs));
    const maxDisp = Math.max(...sensorData.dispX.map(Math.abs));
  
    let diagnosis = "Condição Normal";
  
    if (maxAcc > 1.5 || maxVel > 1000 || maxDisp > 5) {
      diagnosis = "Alta Vibração Detectada!";
    } else if (maxAcc > 1.0 || maxVel > 500 || maxDisp > 2) {
      diagnosis = "Vibração Moderada";
    }
  
    // Obter a classe da máquina selecionada
    const machineClass = parseInt(document.getElementById('machineClass').value);
  
    // Avaliar nível de vibração baseado na norma ISO 10816
    const vibrationLevel = getVibrationZone(maxVel, machineClass);
  
    document.getElementById('diagnosticResult').textContent =
      `Diagnóstico: ${diagnosis}
  Máxima Velocidade: ${maxVel.toFixed(2)} mm/s
  Classificação ISO 10816: ${vibrationLevel}`;
  
    return `${diagnosis} - ${vibrationLevel}`;
  }
  
// Salvar Medição no IndexedDB
function saveMeasurement() {
    const now = new Date().toLocaleString('pt-BR');

    const entry = {
        cliente: clientName,
        equipamento: equipmentName,
        dataHora: now,
        dados: dataBySensor,
        imagem: imageBase64,
        diagnostico: generateDiagnosis(dataBySensor[0])
    };

    const transaction = db.transaction(['historico'], 'readwrite');
    const store = transaction.objectStore('historico');
    const addRequest = store.add(entry);

    addRequest.onsuccess = function() {
        console.log('Medição salva com sucesso.');
    };

    addRequest.onerror = function(event) {
        console.error('Erro ao salvar:', event.target.error);
    };
}

// Carregar Histórico do IndexedDB
function loadHistory() {
    const list = document.getElementById('historyList');
    list.innerHTML = '';

    const transaction = db.transaction(['historico'], 'readonly');
    const store = transaction.objectStore('historico');
    const getAllRequest = store.getAll();

    getAllRequest.onsuccess = function() {
        const history = getAllRequest.result;

        history.forEach((item) => {
            const div = document.createElement('div');
            div.innerHTML = `
                <p><strong>${item.cliente}</strong> - ${item.equipamento} (${item.dataHora})</p>
                <button onclick="visualizeHistory(${item.id})">Visualizar Gráficos</button>
                <button onclick="generatePDF(${item.id})">Gerar PDF</button>
                <button onclick="deleteMeasurement(${item.id})">Excluir</button>
                <hr>
            `;
            list.appendChild(div);
        });
    };
}

// Visualizar uma medição histórica
function visualizeHistory(id) {
    const transaction = db.transaction(['historico'], 'readonly');
    const store = transaction.objectStore('historico');
    const getRequest = store.get(id);

    getRequest.onsuccess = function() {
        const item = getRequest.result;
        const dados = item.dados[0];

        plotAcceleration(dados);
        plotVelocity(dados);
        plotDisplacement(dados);
        plotFFT(dados);

        document.getElementById('diagnosticResult').textContent = item.diagnostico;
    };
}

// Excluir uma medição do IndexedDB
function deleteMeasurement(id) {
    const transaction = db.transaction(['historico'], 'readwrite');
    const store = transaction.objectStore('historico');
    const deleteRequest = store.delete(id);

    deleteRequest.onsuccess = function() {
        loadHistory();
    };

    deleteRequest.onerror = function(event) {
        console.error('Erro ao excluir:', event.target.error);
    };
}
// Exportar Dados para XLSX
function exportXLSX() {
    if (dataBySensor.length === 0 || !dataBySensor[0].accX.length) {
        alert("Nenhum dado para exportar!");
        return;
    }

    const exportData = dataBySensor[0].accX.map((_, i) => ({
        Tempo: i,
        AccX: dataBySensor[0].accX[i],
        AccY: dataBySensor[0].accY[i],
        AccZ: dataBySensor[0].accZ[i],
        VelX: dataBySensor[0].velX[i],
        VelY: dataBySensor[0].velY[i],
        VelZ: dataBySensor[0].velZ[i],
        DispX: dataBySensor[0].dispX[i],
        DispY: dataBySensor[0].dispY[i],
        DispZ: dataBySensor[0].dispZ[i]
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, "DadosVibracao");

    XLSX.writeFile(wb, `dados_vibracao_${clientName}_${equipmentName}.xlsx`);
}

// Importar Dados de XLSX/CSV
function importData() {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.xlsx, .csv';

    input.onchange = e => {
        const file = e.target.files[0];
        const reader = new FileReader();

        reader.onload = (evt) => {
            const data = evt.target.result;
            const workbook = XLSX.read(data, { type: 'binary' });

            const firstSheet = workbook.SheetNames[0];
            const worksheet = workbook.Sheets[firstSheet];
            const importedData = XLSX.utils.sheet_to_json(worksheet);

            // Preencher os arrays para plotar
            const sensor = {
                accX: [],
                accY: [],
                accZ: [],
                velX: [],
                velY: [],
                velZ: [],
                dispX: [],
                dispY: [],
                dispZ: []
            };

            importedData.forEach(row => {
                sensor.accX.push(parseFloat(row.AccX) || 0);
                sensor.accY.push(parseFloat(row.AccY) || 0);
                sensor.accZ.push(parseFloat(row.AccZ) || 0);
                sensor.velX.push(parseFloat(row.VelX) || 0);
                sensor.velY.push(parseFloat(row.VelY) || 0);
                sensor.velZ.push(parseFloat(row.VelZ) || 0);
                sensor.dispX.push(parseFloat(row.DispX) || 0);
                sensor.dispY.push(parseFloat(row.DispY) || 0);
                sensor.dispZ.push(parseFloat(row.DispZ) || 0);
            });

            dataBySensor = [sensor];

            plotAll(0);
            generateDiagnosis(sensor);

            alert("Dados importados com sucesso!");
        };

        reader.readAsBinaryString(file);
    };

    input.click();
}



// Função para retornar a zona de vibração conforme a classe
function getVibrationZone(vibration, machineClass) {
    const limits = vibration_limits[machineClass - 1];
  
    if (vibration <= limits[0]) return "Zona A (Excelente)";
    if (vibration <= limits[1]) return "Zona B (Aceitável)";
    if (vibration <= limits[2]) return "Zona C (Não Aceitável)";
    return "Zona D (Crítico)";
  }

  

// Previsão de Falha via IA (Simulada)
async function predictFailure() {
    if (dataBySensor.length === 0 || !dataBySensor[0].accX.length) {
        alert("Nenhum dado disponível para previsão!");
        return;
    }

    const fftX = fftSimple(dataBySensor[0].accX);
    const fftY = fftSimple(dataBySensor[0].accY);
    const fftZ = fftSimple(dataBySensor[0].accZ);

    const inputData = {
        fft_x: fftX.slice(0, 128),
        fft_y: fftY.slice(0, 128),
        fft_z: fftZ.slice(0, 128)
    };

    try {
        // Simulando chamada para uma IA (mock)
        const response = await fetch('https://mockapi.io/predict-failure', {
            method: 'POST',
            mode: 'no-cors',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(inputData)
        });

        
        if (!response.ok) {
            throw new Error("Erro na comunicação com IA");
        }

        const result = await response.json();
        alert(`Previsão de Falha: ${result.prediction}`);

    } catch (error) {
        console.error(error);
        alert("Erro ao prever falha: " + error.message);
    }
}

async function generatePDF(id) {
    const { jsPDF } = window.jspdf;
    const doc = new jsPDF();

    const transaction = db.transaction(['historico'], 'readonly');
    const store = transaction.objectStore('historico');
    const getRequest = store.get(id);

    getRequest.onsuccess = async function() {
        const item = getRequest.result;

        doc.setFontSize(16);
        doc.text(`Relatório de Vibração`, 10, 10);
        doc.setFontSize(12);
        doc.text(`Cliente: ${item.cliente}`, 10, 20);
        doc.text(`Equipamento: ${item.equipamento}`, 10, 30);
        doc.text(`Data: ${item.dataHora}`, 10, 40);
        doc.text(`Diagnóstico: ${item.diagnostico}`, 10, 50);

        if (item.imagem) {
            doc.addImage(item.imagem, 'JPEG', 130, 10, 60, 45);
        }

        const canvasIds = ['accelerationChart', 'velocityChart', 'displacementChart', 'fftChartX', 'fftChartY', 'fftChartZ'];

        for (const id of canvasIds) {
            const canvas = document.getElementById(id);
            if (canvas) {
                const imgData = canvas.toDataURL('image/jpeg', 1.0);
                doc.addPage();
                doc.text(`Gráfico ${id}`, 10, 10);
                doc.addImage(imgData, 'JPEG', 10, 20, 180, 100);
            }
        }

        doc.save(`Relatorio_${item.cliente}_${item.equipamento}_${item.dataHora.replace(/[/:]/g, '-')}.pdf`);
    };
}
