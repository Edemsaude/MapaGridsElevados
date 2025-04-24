<!DOCTYPE html>
<html>
<head>
  <title>Mapa de Grids - Campo Grande/MS</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    body { 
      font-family: Arial, sans-serif; 
      margin: 15px;
      padding: 0;
    }
    #map { 
      height: 70vh; 
      width: 100%;
      margin-top: 15px; 
      border: 1px solid #ddd; 
    }
    .control-panel { 
      margin-bottom: 15px; 
    }
    .filter-container {
      margin: 10px 0;
      display: flex;
      align-items: center;
    }
    #distritoFilter {
      flex: 1;
      padding: 8px;
      margin-right: 5px;
    }
    button { 
      padding: 8px 12px; 
      margin: 5px 5px 5px 0; 
      cursor: pointer; 
      font-size: 14px;
    }
    input[type="file"] { 
      margin-bottom: 10px; 
      width: 100%;
    }
    #status { 
      padding: 10px; 
      margin: 10px 0; 
      border-radius: 4px; 
      background: #f0f0f0;
      font-size: 14px;
    }
    .search-container {
      margin: 10px 0;
      display: flex;
    }
    #searchInput {
      flex: 1;
      padding: 8px;
      margin-right: 5px;
    }
    .legend {
      padding: 6px 8px;
      background: white;
      border-radius: 5px;
      box-shadow: 0 0 15px rgba(0,0,0,0.2);
      line-height: 1.5;
      font-size: 12px;
    }
    .legend i {
      width: 12px;
      height: 12px;
      display: inline-block;
      margin-right: 5px;
      vertical-align: middle;
    }
    .grid-label {
      background: rgba(255,255,255,0.8);
      border-radius: 3px;
      padding: 2px 4px;
      border: 1px solid #333;
      font-size: 10px;
      font-weight: bold;
      pointer-events: none;
      white-space: nowrap;
    }
    .custom-tooltip {
      font-weight: bold;
      font-size: 14px;
    }
    .highlighted {
      animation: pulse 0.5s infinite alternate;
      stroke: #0055ff;
      stroke-width: 3;
    }
    @keyframes pulse {
      from { transform: scale(1); }
      to { transform: scale(1.2); }
    }
    @media (max-width: 600px) {
      #map { height: 60vh; }
      .filter-container { flex-direction: column; }
      #distritoFilter { margin-right: 0; margin-bottom: 5px; }
      .search-container { flex-direction: column; }
      #searchInput { margin-right: 0; margin-bottom: 5px; }
    }
  </style>
</head>
<body>
  <div class="control-panel">
    <h2>Mapa de Grids Elevados</h2>
    <input type="file" id="fileInput" accept=".xlsx,.xls,.csv">
    <button onclick="carregarDados()">Carregar Excel</button>
    
    <div class="filter-container">
      <select id="distritoFilter" disabled>
        <option value="">Selecione um Distrito</option>
        <option value="ANHANDUI 1">ANHANDUI 1</option>
        <option value="ANHANDUI 2">ANHANDUI 2</option>
        <option value="BANDEIRA">BANDEIRA</option>
        <option value="CENTRO">CENTRO</option>
        <option value="IMBIRUSSU">IMBIRUSSU</option>
        <option value="LAGOA">LAGOA</option>
        <option value="PROSA">PROSA</option>
        <option value="SEGREDO">SEGREDO</option>
      </select>
      <button id="plotBtn" disabled onclick="plotarDistrito()">Plotar Distrito</button>
    </div>
    
    <div class="search-container">
      <input type="text" id="searchInput" placeholder="Digite o código do grid (ex: 1198)">
      <button onclick="buscarGrid()">Buscar</button>
    </div>
    
    <div id="status">Selecione um arquivo Excel para começar</div>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
  <script>
    // Variáveis globais
    let map;
    let dadosCompletos = [];
    let currentMarkers = [];
    const CG_CENTER = [-20.4697, -54.6201];
    const OVOS_MINIMOS = 50;

    // Inicializa o mapa
    function initMap() {
      if (!map) {
        map = L.map('map').setView(CG_CENTER, 12);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
        }).addTo(map);
        
        // Adiciona legenda
        const legend = L.control({position: 'bottomright'});
        legend.onAdd = function() {
          const div = L.DomUtil.create('div', 'legend');
          div.innerHTML = `
            <h4>Legenda:</h4>
            <div><i style="background:#ffcc00"></i> 50-100 ovos</div>
            <div><i style="background:#ff6600"></i> 101-200 ovos</div>
            <div><i style="background:#cc0000"></i> ≥201 ovos</div>
          `;
          return div;
        };
        legend.addTo(map);
      }
      return map;
    }

    // Carrega os dados do Excel
    async function carregarDados() {
      const fileInput = document.getElementById('fileInput');
      const statusDiv = document.getElementById('status');
      
      if (!fileInput.files.length) {
        statusDiv.textContent = 'Nenhum arquivo selecionado';
        statusDiv.style.backgroundColor = '#f8d7da';
        return;
      }

      const file = fileInput.files[0];
      statusDiv.textContent = `Processando ${file.name}...`;
      statusDiv.style.backgroundColor = '#fff3cd';

      try {
        const data = await readFile(file);
        const workbook = XLSX.read(data, { type: 'array' });
        const sheetName = workbook.SheetNames[0];
        const sheet = workbook.Sheets[sheetName];
        
        dadosCompletos = XLSX.utils.sheet_to_json(sheet, { defval: "" })
          .filter(row => {
            const ovos = parseInt(row.CONT_OVOS || row.OVOS || 0);
            return ovos >= OVOS_MINIMOS;
          })
          .map(row => ({
            grid: (row.COD_GRID || row.GRID || '').toString().trim(),
            ovos: parseInt(row.CONT_OVOS || row.OVOS || 0),
            endereco: row.LOGRADOURO || row.ENDEREÇO || '',
            distrito: (row.NM_DISTRITO || row.DISTRITO || '').toUpperCase().trim()
          }));

        // Habilita o seletor de distritos
        document.getElementById('distritoFilter').disabled = false;
        document.getElementById('plotBtn').disabled = false;
        
        statusDiv.textContent = `Dados carregados: ${dadosCompletos.length} registros com ≥${OVOS_MINIMOS} ovos`;
        statusDiv.style.backgroundColor = '#d4edda';
        
        // Inicializa o mapa se ainda não estiver inicializado
        initMap();
        
      } catch (error) {
        console.error("Erro:", error);
        statusDiv.textContent = `Erro: ${error.message}`;
        statusDiv.style.backgroundColor = '#f8d7da';
      }
    }

    // Plota apenas os grids do distrito selecionado
    async function plotarDistrito() {
      const distritoSelect = document.getElementById('distritoFilter');
      const distrito = distritoSelect.value.toUpperCase().trim();
      const statusDiv = document.getElementById('status');
      
      if (!distrito || distrito === "") {
        statusDiv.textContent = 'Selecione um distrito primeiro';
        statusDiv.style.backgroundColor = '#f8d7da';
        return;
      }

      statusDiv.textContent = `Plotando grids do distrito ${distrito}...`;
      statusDiv.style.backgroundColor = '#fff3cd';

      // Limpa o mapa completamente
      const mapa = initMap();
      clearMarkers();

      const dadosFiltrados = dadosCompletos.filter(item => 
        item.distrito === distrito
      );
      
      if (dadosFiltrados.length === 0) {
        statusDiv.textContent = `Nenhum grid encontrado no distrito ${distrito}`;
        statusDiv.style.backgroundColor = '#f8d7da';
        return;
      }

      let count = 0;
      for (const item of dadosFiltrados) {
        try {
          const coords = await geocodeAddress(item.endereco + ', Campo Grande MS');
          if (coords) {
            // Cria o marcador com círculo colorido
            const marker = L.circleMarker([coords.lat, coords.lng], {
              radius: 18,
              fillColor: getColorForOvos(item.ovos),
              color: '#000',
              fillOpacity: 0.8,
              weight: 1
            }).addTo(mapa).bindPopup(`
              <div class="custom-tooltip">
                <b>Grid:</b> ${item.grid}<br>
                <b>Ovos:</b> ${item.ovos}<br>
                <b>Endereço:</b> ${item.endereco}<br>
                <b>Distrito:</b> ${item.distrito}
              </div>
            `);
            
            // Adiciona label
            const label = L.marker([coords.lat, coords.lng], {
              icon: L.divIcon({
                className: 'grid-label',
                html: item.grid,
                iconSize: null,
                iconAnchor: [0, 0]
              }),
              zIndexOffset: 1000,
              interactive: false
            }).addTo(mapa);
            
            // Armazena o código do grid como propriedade do marcador
            marker.gridCode = item.grid;
            currentMarkers.push(marker);
            count++;
          }
        } catch (error) {
          console.error(`Erro no endereço: ${item.endereco}`, error);
        }
      }

      statusDiv.textContent = `${count} grids plotados no distrito ${distrito}`;
      statusDiv.style.backgroundColor = '#d4edda';
      
      // Ajusta o zoom para mostrar todos os markers
      if (count > 0) {
        const group = new L.featureGroup(currentMarkers);
        mapa.fitBounds(group.getBounds(), { padding: [50, 50] });
      }
    }

    // Busca grid no mapa com efeitos visuais
    function buscarGrid() {
      const searchTerm = document.getElementById('searchInput').value.trim();
      const statusDiv = document.getElementById('status');
      
      if (!searchTerm) {
        statusDiv.textContent = 'Digite um código de grid para buscar';
        statusDiv.style.backgroundColor = '#f8d7da';
        return;
      }

      // Remove destaque anterior
      currentMarkers.forEach(marker => {
        marker.setStyle({
          color: '#000',
          weight: 1
        });
        if (marker._path) {
          marker._path.classList.remove('highlighted');
        }
      });

      // Filtra os marcadores visíveis
      const foundMarkers = currentMarkers.filter(marker => 
        marker.gridCode && marker.gridCode.toString().includes(searchTerm)
      );

      if (foundMarkers.length === 0) {
        statusDiv.textContent = `Nenhum grid encontrado com "${searchTerm}"`;
        statusDiv.style.backgroundColor = '#f8d7da';
        return;
      }

      // Destaca os marcadores encontrados
      foundMarkers.forEach(marker => {
        marker.setStyle({
          color: '#0055ff',
          weight: 3
        });
        if (marker._path) {
          marker._path.classList.add('highlighted');
        }
        marker.openPopup();
      });

      // Centraliza no primeiro marcador encontrado
      if (foundMarkers.length > 0) {
        if (foundMarkers.length === 1) {
          map.setView(foundMarkers[0].getLatLng(), 17);
        } else {
          const group = new L.featureGroup(foundMarkers);
          map.fitBounds(group.getBounds(), { padding: [50, 50] });
        }
        
        statusDiv.textContent = `Encontrado(s) ${foundMarkers.length} grid(s) com "${searchTerm}"`;
        statusDiv.style.backgroundColor = '#d4edda';
      }
    }

    // Limpa todos os markers do mapa
    function clearMarkers() {
      if (!map) return;
      map.eachLayer(layer => {
        if (layer instanceof L.Marker || layer instanceof L.CircleMarker) {
          map.removeLayer(layer);
        }
      });
      currentMarkers = [];
    }

    // Função auxiliar para determinar a cor baseada na quantidade de ovos
    function getColorForOvos(ovos) {
      const num = parseInt(ovos) || 0;
      if (num >= 201) return '#cc0000';
      if (num >= 101) return '#ff6600';
      return '#ffcc00';
    }

    // Função auxiliar para ler arquivo
    function readFile(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = e => resolve(e.target.result);
        reader.onerror = e => reject(new Error('Falha ao ler arquivo'));
        reader.readAsArrayBuffer(file);
      });
    }

    // Função auxiliar para geocodificação
    async function geocodeAddress(address) {
      try {
        const response = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(address)}&limit=1`);
        const data = await response.json();
        if (!data || data.length === 0) return null;
        return { lat: parseFloat(data[0].lat), lng: parseFloat(data[0].lon) };
      } catch (error) {
        console.error('Erro na geocodificação:', error);
        return null;
      }
    }

    // Inicializa o mapa quando a página carrega
    window.onload = function() {
      initMap();
      // Adiciona evento para tecla Enter no campo de busca
      document.getElementById('searchInput').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') buscarGrid();
      });
    };
  </script>
</body>
</html>
