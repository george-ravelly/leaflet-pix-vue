<template>
  <div class="app-container">
    <div class="controls-panel">
      <h2>Carregar GeoJSON</h2>
      <input type="file" @change="handleFileUpload" accept=".geojson,.json" />
      <button @click="clearExternalGeojson" :disabled="!myExternalGeojson">Limpar Camada Externa</button> <br />
      <p v-if="uploadError" class="error-message">{{ uploadError }}</p>
      <p v-if="myExternalGeojson" class="success-message">Arquivo GeoJSON carregado com sucesso!</p>
    </div>

    <!-- <MapsPixie :dataGeojson="myExternalGeojson" /> -->
    <HelloWorld :dataGeojson="myExternalGeojson" />
  </div>
</template>

<script setup>
import { ref } from 'vue';
import MapsPixie from './components/MapsPixie.vue'; // Ajuste o caminho se necessário
import HelloWorld from './components/HelloWorld.vue';

const myExternalGeojson = ref(null);
const uploadError = ref('');

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) {
    uploadError.value = 'Nenhum arquivo selecionado.';
    return;
  }

  // Verifica se o arquivo é um JSON ou GeoJSON
  if (!file.type.includes('json') && !file.name.endsWith('.geojson')) {
    uploadError.value = 'Por favor, selecione um arquivo .json ou .geojson válido.';
    myExternalGeojson.value = null;
    return;
  }

  uploadError.value = ''; // Limpa erros anteriores

  const reader = new FileReader();

  reader.onload = (e) => {
    try {
      const parsedData = JSON.parse(e.target.result);
      // Opcional: Adicione uma validação básica para GeoJSON
      if (parsedData.type !== 'FeatureCollection' && parsedData.type !== 'Feature' && parsedData.type !== 'Polygon' && parsedData.type !== 'MultiPolygon') {
        throw new Error('O arquivo JSON não parece ser um GeoJSON válido (tipo FeatureCollection, Feature ou Polygon/MultiPolygon esperado).');
      }
      myExternalGeojson.value = parsedData;
    } catch (error) {
      console.error("Erro ao ler ou parsear o arquivo GeoJSON:", error);
      uploadError.value = `Erro ao carregar o arquivo: ${error.message}`;
      myExternalGeojson.value = null;
    }
  };

  reader.onerror = () => {
    uploadError.value = 'Erro ao ler o arquivo.';
    myExternalGeojson.value = null;
  };

  reader.readAsText(file); // Lê o arquivo como texto
};

const clearExternalGeojson = () => {
  myExternalGeojson.value = null;
  uploadError.value = '';
  // Opcional: Resetar o input de arquivo para que o mesmo arquivo possa ser selecionado novamente
  const fileInput = document.querySelector('input[type="file"]');
  if (fileInput) {
    fileInput.value = '';
  }
};
</script>

<style>
.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100vw;
  overflow: hidden; /* Evita scrollbars */
}

.controls-panel {
  display: flex;
  padding: 15px;
  background-color: #f8f8f8;
  border-bottom: 1px solid #eee;
  align-items: center;
  gap: 15px;
  z-index: 1000; /* Garante que o painel de controle esteja acima do mapa */
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.controls-panel h2 {
  margin: 0;
  font-size: 1.2em;
  color: #333;
}

.controls-panel input[type="file"] {
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 5px;
  background-color: #fff;
  cursor: pointer;
}

.controls-panel button {
  padding: 8px 15px;
  border: none;
  border-radius: 5px;
  background-color: #007bff;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.controls-panel button:hover:not(:disabled) {
  background-color: #0056b3;
}

.controls-panel button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.error-message {
  color: #dc3545;
  font-size: 0.9em;
}

.success-message {
  color: #28a745;
  font-size: 0.9em;
}
</style>
