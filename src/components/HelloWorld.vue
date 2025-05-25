<template>
  <div id="map" ref="mapRef"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount, defineProps, watch } from 'vue';
import * as L from 'leaflet';
import 'leaflet/dist/leaflet.css';

import * as PIXI from 'pixi.js';

// Importante: garante que PIXI esteja global para leaflet-pixi-overlay
if (typeof window.PIXI === 'undefined' || window.PIXI === null || typeof window.PIXI.VERSION === 'undefined') {
  window.PIXI = PIXI;
}

import 'leaflet-pixi-overlay';

// --- Props do Componente ---
const props = defineProps({
  dataGeojson: {
    type: Object,
    default: null
  }
});

const mapRef = ref(null);
let map = null;
let overlay = null;
const dynamicGeojsonLayer = ref(null);

// --- GeoJSONs Padrão (Camadas Internas) ---
const geojsonNatalInteiroDefault = {
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "name": "Natal Inteiro", "color": 0x888888, "alpha": 0.2 },
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [-35.30, -5.90],
            [-35.30, -5.65],
            [-35.15, -5.65],
            [-35.15, -5.90],
            [-35.30, -5.90]
          ]
        ]
      }
    }
  ]
};

const geojsonPontosNatalDefault = {
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "name": "Ponta Negra", "color": 0xff0000 },
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [-35.20, -5.88],
            [-35.20, -5.85],
            [-35.18, -5.85],
            [-35.18, -5.88],
            [-35.20, -5.88]
          ]
        ]
      }
    },
    {
      "type": "Feature",
      "properties": { "name": "Centro", "color": 0x0000ff },
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [-35.22, -5.79],
            [-35.22, -5.77],
            [-35.20, -5.77],
            [-35.20, -5.79],
            [-35.22, -5.79]
          ]
        ]
      }
    }
  ]
};

// Variável para o PIXI.Ticker
let ticker = null;
// Variável para a cor animada
let animatedColor = 0xff0000;

onMounted(() => {
  map = L.map(mapRef.value).setView([-5.7945, -35.211], 12);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: 'Map data © OpenStreetMap contributors'
  }).addTo(map);

  const pixiContainer = new PIXI.Container();

  overlay = L.pixiOverlay(function(utils) {
    const container = utils.getContainer();
    const renderer = utils.getRenderer();
    const project = utils.latLngToLayerPoint;
    const scale = utils.getScale();

    // --- CORREÇÃO DO VAZAMENTO DE MEMÓRIA ---
    // Destrói os filhos existentes antes de removê-los e criar novos.
    // Isso libera os recursos WebGL associados a esses objetos.
    while (container.children.length > 0) {
      const child = container.children[0];
      // Destrói o filho e seus recursos (texturas, baseTextures, geometria)
      child.destroy({ children: true, texture: true, baseTexture: true, geometry: true });
      // removeChild() não é estritamente necessário após destroy(),
      // mas removeChildren() abaixo fará a limpeza da lista.
    }
    container.removeChildren(); // Limpa o container após destruir os filhos

    function getRandomColor() {
      // Gera um inteiro entre 0x000000 e 0xFFFFFF
      return Math.floor(Math.random() * 0xFFFFFF);
    }

    // --- Função auxiliar para desenhar uma coleção de features ---
    const drawFeatures = (features, defaultColor = 0xAAAAAA, defaultAlpha = 0.5) => {
      if (!features) return;

      features.forEach(feature => {
        if (!feature.geometry || !feature.geometry.coordinates || feature.geometry.coordinates.length === 0) {
          console.warn("Skipping feature without geometry or coordinates:", feature);
          return;
        }
        
        let tempDefault = null;
        // Usa a cor e alfa da feature.properties, ou defaults
        if (feature.properties.color === undefined) {
          tempDefault = getRandomColor();
        }

        const graphics = new PIXI.Graphics();
        graphics.lineStyle(0.5 / scale, 0x333333, 1);
        graphics.beginFill(feature.properties.color || (tempDefault || defaultColor), feature.properties.alpha || defaultAlpha);

        let polygonsToDraw = [];
        if (feature.geometry.type === "Polygon") {
          polygonsToDraw.push(feature.geometry.coordinates);
        } else if (feature.geometry.type === "MultiPolygon") {
          polygonsToDraw = feature.geometry.coordinates;
        } else {
          console.warn(`Skipping unsupported geometry type: ${feature.geometry.type}`, feature);
          return;
        }

        polygonsToDraw.forEach(polygonCoords => {
          const outerRing = polygonCoords[0];
          const pixiPoints = outerRing.map(c => {
            const projected = project(L.latLng(c[1], c[0]));
            return [projected.x, projected.y];
          }).flat();
          graphics.drawPolygon(pixiPoints);

          for (let i = 1; i < polygonCoords.length; i++) {
            const holeRing = polygonCoords[i];
            const holePixiPoints = holeRing.map(c => {
              const projected = project(L.latLng(c[1], c[0]));
              return [projected.x, projected.y];
            }).flat();
            graphics.drawPolygon(holePixiPoints);
          }
        });

        graphics.endFill();

        graphics.interactive = true;
        graphics.buttonMode = true;
        graphics.on('pointerdown', () => {
          alert('Clicou no polígono: ' + feature.properties.name);
        });

        container.addChild(graphics);
      });
    };

    // --- Desenha as Camadas Padrão ---
    drawFeatures(geojsonNatalInteiroDefault.features);
    drawFeatures(geojsonPontosNatalDefault.features);

    // --- Desenha a Camada Dinâmica (recebida via prop) ---
    drawFeatures(dynamicGeojsonLayer.value?.features, 0xFFD700, 0.7);

    renderer.render(container);
  }, pixiContainer);

  overlay.addTo(map);

  // Observa a prop 'dataGeojson' para atualizar a camada dinâmica
  watch(() => props.dataGeojson, (newVal) => {
    dynamicGeojsonLayer.value = newVal;
    if (overlay) {
      overlay.redraw();
    }
  }, { immediate: true });

  // --- Configuração do PIXI.Ticker para animação ---
  ticker = PIXI.Ticker.shared; // Usa o ticker compartilhado do PIXI
  ticker = new PIXI.Ticker();
  let lastUpdate = 0;
  ticker.maxFPS = 12;
  ticker.add((delta) => {
    // Atualiza a cada 2 segundos (2000 ms)
    lastUpdate += ticker.elapsedMS;
    if (lastUpdate >= 2000) {
      if (dynamicGeojsonLayer.value?.features?.[0]) {
        // dynamicGeojsonLayer.value.features[0].properties.color = 0x00ff00;
        if (overlay) overlay.redraw();
        console.log("Cor do polígono 'Ponta Negra' alterada para verde e overlay redesenhado.");
      }
      lastUpdate = 0;
    }
  });
  ticker.start();
});

onBeforeUnmount(() => {
  if (map) {
    map.remove(); // Limpa o mapa ao desmontar o componente
  }
  // --- PARAR O TICKER E REMOVER O LISTENER ---
  if (ticker) {
    ticker.stop(); // Para o ticker
    ticker.remove(animatedColor); // Remove o listener para evitar vazamentos
  }
});
</script>

<style>
#map {
  width: 100vw;
  height: 100vh;
  position: relative;
  background-color: #f0f0f0;
}
html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  overflow: hidden;
}
</style>
