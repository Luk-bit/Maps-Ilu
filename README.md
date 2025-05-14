# Maps-Ilu
Mapa de iluminacao Carmo da Cachoeira MG

<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <title>Mapa de Iluminação Pública</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.css" />
  <style>
    html, body, #map {
      height: 100%;
      margin: 0;
    }
    #admin-login, #logout, #downloadExcel {
      position: absolute;
      right: 10px;
      z-index: 9999;
      background: white;
      border: 1px solid #ccc;
      border-radius: 5px;
      padding: 6px 10px;
      font-size: 14px;
      cursor: pointer;
    }
    #admin-login { top: 10px; }
    #logout { top: 50px; display: none; }
    #downloadExcel { top: 90px; }
  </style>
</head>
<body>
  <div id="admin-login">🔐 Login</div>
  <div id="logout">🚪 Logout</div>
  <div id="downloadExcel">⬇️ Excel</div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>

  <script>
    const map = L.map('map').setView([-23.55, -46.63], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

    let isAdmin = localStorage.getItem('isAdmin') === 'true';
    let postes = JSON.parse(localStorage.getItem('postes') || '[]');
    let rotaAtual = null;
    let marcadorLocal = null;

    document.getElementById('admin-login').onclick = () => {
      const user = prompt("Usuário administrador:");
      const senha = prompt("Digite a senha do administrador:");
      if (user === "admin" && senha === "admin123") {
        isAdmin = true;
        localStorage.setItem('isAdmin', true);
        alert("Modo administrador ativado.");
        document.getElementById('logout').style.display = 'block';
        renderizarPostes();
      } else {
        alert("Credenciais inválidas.");
      }
    };

    document.getElementById('logout').onclick = () => {
      isAdmin = false;
      localStorage.removeItem('isAdmin');
      document.getElementById('logout').style.display = 'none';
      alert("Você saiu do modo administrador.");
      renderizarPostes();
    };

    document.getElementById('downloadExcel').onclick = () => {
      const dados = postes.map(p => ({
        Rua: p.rua || '',
        Bairro: p.bairro || '',
        Código: p.codigo || '',
        Status: p.status,
        'Última alteração': p.ultimaAlteracao
      }));
      const wb = XLSX.utils.book_new();
      const ws = XLSX.utils.json_to_sheet(dados);
      XLSX.utils.book_append_sheet(wb, ws, 'Postes');
      XLSX.writeFile(wb, 'postes.xlsx');
    };

    function adicionarPoste(latlng) {
      if (!isAdmin) return alert("Apenas administradores podem adicionar postes.");
      const rua = prompt("Digite a rua:");
      const bairro = prompt("Digite o bairro:");
      const codigo = prompt("Digite o código do poste:");
      if (!rua || !bairro || !codigo) return alert("Preencha todos os campos.");

      const poste = {
        id: Date.now(),
        lat: latlng.lat,
        lng: latlng.lng,
        rua,
        bairro,
        codigo,
        status: 'apagada',
        ultimaAlteracao: new Date().toLocaleString(),
        imagem: ''
      };
      postes.push(poste);
      salvarPostes();
      renderizarPostes();
    }

    function salvarPostes() {
      localStorage.setItem('postes', JSON.stringify(postes));
    }

    function excluirPoste(id) {
      postes = postes.filter(p => p.id !== id);
      salvarPostes();
      renderizarPostes();
    }

    function alterarStatusPorId(id, status) {
      const poste = postes.find(p => p.id === id);
      if (!poste) return;
      if (!isAdmin && status === 'acesa') return alert("Visitantes não podem ativar postes.");
      alterarStatus(poste, status);
    }

    function alterarStatus(poste, status) {
      poste.status = status;
      poste.ultimaAlteracao = new Date().toLocaleString();
      salvarPostes();
      renderizarPostes();
    }

    function desenharRota(destino) {
      if (rotaAtual) map.removeControl(rotaAtual);
      if (!navigator.geolocation) return alert("Geolocalização não suportada.");

      navigator.geolocation.getCurrentPosition(pos => {
        const origem = L.latLng(pos.coords.latitude, pos.coords.longitude);
        rotaAtual = L.Routing.control({
          waypoints: [origem, L.latLng(destino[0], destino[1])],
          routeWhileDragging: false,
          createMarker: () => null
        }).addTo(map);
      }, () => {
        alert("Erro ao obter localização.");
      });
    }

    function adicionarImagem(id) {
      const input = document.createElement('input');
      input.type = 'file';
      input.accept = 'image/*';
      input.capture = 'environment';
      input.onchange = () => {
        const file = input.files[0];
        if (!file) return;
        const reader = new FileReader();
        reader.onload = e => {
          const poste = postes.find(p => p.id === id);
          poste.imagem = e.target.result;
          salvarPostes();
          renderizarPostes();
        };
        reader.readAsDataURL(file);
      };
      input.click();
    }

    function renderizarPostes() {
      map.eachLayer(layer => {
        if (layer instanceof L.Marker || layer instanceof L.CircleMarker) {
          map.removeLayer(layer);
        }
      });

      if (marcadorLocal) marcadorLocal.remove();

      navigator.geolocation.getCurrentPosition(pos => {
        const userLatLng = L.latLng(pos.coords.latitude, pos.coords.longitude);
        marcadorLocal = L.marker(userLatLng).addTo(map).bindPopup("Você está aqui").openPopup();
      });

      postes.forEach(poste => {
        const cor = poste.status === 'acesa' ? 'green' : poste.status === 'apagada' ? 'gray' : 'orange';

        const marker = L.circleMarker([poste.lat, poste.lng], {
          radius: 10,
          color: cor,
          fillColor: cor,
          fillOpacity: 0.8
        }).addTo(map);

        let botoes = `
          <button onclick="alterarStatusPorId(${poste.id}, 'apagada')">Apagada</button>
          <button onclick="alterarStatusPorId(${poste.id}, 'problema')">Problema</button>
        `;
        if (isAdmin) {
          botoes += `<button onclick="alterarStatusPorId(${poste.id}, 'acesa')">Acesa</button>`;
          botoes += `<button onclick="excluirPoste(${poste.id})">🗑️ Excluir</button>`;
        }

        botoes += `<button onclick="adicionarImagem(${poste.id})">🖼️ Imagem</button>`;
        botoes += `<button onclick="desenharRota([${poste.lat}, ${poste.lng}])">🧭 Ver rota</button>`;

        const popup = `
          <b>${poste.codigo}</b><br>${poste.rua}, ${poste.bairro}<br>Status: ${poste.status}<br>
          Última alteração: ${poste.ultimaAlteracao}<br>
          ${poste.imagem ? `<img src="${poste.imagem}" width="100">` : ''}
          <br>${botoes}
        `;
        marker.bindPopup(popup);
      });
    }

    map.on('click', e => {
      if (!isAdmin) return alert("Apenas administradores podem adicionar postes.");
      adicionarPoste(e.latlng);
    });

    // Inicial
    renderizarPostes();
    if (isAdmin) document.getElementById('logout').style.display = 'block';

    // Atualiza marcador a cada 10 segundos
    setInterval(renderizarPostes, 10000);
  </script>
</body>
</html>
