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
// Define a prop 'dataGeojson' que pode receber um objeto GeoJSON
const props = defineProps({
  dataGeojson: {
    type: Object,
    default: null // Opcional: pode ser nulo se nenhum GeoJSON for fornecido
  }
});

const mapRef = ref(null);
let map = null;
let overlay = null;
// Ref para o GeoJSON que será exibido via prop
const dynamicGeojsonLayer = ref(null);

// Função para limpar o Graphics
function clearGraphics(graphics) {
    // Limpa o conteúdo do Graphics
    graphics.clear();

    // Remove o Graphics do palco
    app.stage.removeChild(graphics);

    // Remove todos os ouvintes de eventos
    graphics.removeAllListeners();

    // Destroi o objeto Graphics
    graphics.destroy();
}

// --- GeoJSONs Padrão (Camadas Internas) ---
// GeoJSON para representar a área de Natal/RN (camada padrão 1)
const geojsonNatalInteiroDefault = {
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "name": "Natal Inteiro", "color": 0x888888, "alpha": 0.2 }, // Cinza claro, semi-transparente
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

// GeoJSON para representar alguns pontos/zonas específicas de Natal (camada padrão 2)
const geojsonPontosNatalDefault = {
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": { "name": "Ponta Negra", "color": 0xff0000 }, // Cor inicial: vermelho
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
      "properties": { "name": "Centro", "color": 0x0000ff }, // Cor inicial: azul
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

let ticker = null;

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

    container.children.forEach(child => {
      if (child.destroy) {
        child.clear()
        // clearGraphics(child)
        child.removeAllListeners()
        child.destroy({ children: true, texture: true, baseTexture: true })
      };
    });
    container.removeChildren();
    function getRandomColor() {
      // Gera um inteiro entre 0x000000 e 0xFFFFFF
      return Math.floor(Math.random() * 0xFFFFFF);
    }

    // --- Função auxiliar para desenhar uma coleção de features ---
    const drawFeatures = (features, defaultColor = 0xAAAAAA, defaultAlpha = 0.5, name = undefined) => {
      if (!features) {
        return
      }; // Garante que features existe

      features.forEach(feature => {
        // Ignora features que não são Polygon por simplicidade neste exemplo
        let polygonsToDraw = [];

        if (feature.geometry.type === "Polygon") {
          polygonsToDraw.push(feature.geometry.coordinates);
        } else if (feature.geometry.type === "MultiPolygon") {
          polygonsToDraw = feature.geometry.coordinates;
        } else {
          console.warn(`Skipping unsupported geometry type: ${feature.geometry.type}`, feature);
          return; // Ignora outros tipos de geometria por enquanto
        }

        polygonsToDraw.forEach(polygonCoords => {

          const graphics = new PIXI.Graphics();
          graphics.lineStyle(2 / scale, 0x333333, 1);
          let tempDefault = null;
          // Usa a cor e alfa da feature.properties, ou defaults
          if (feature.properties.color === undefined) {
            tempDefault = getRandomColor();
          }
          graphics.beginFill(feature.properties.color || (tempDefault || defaultColor), feature.properties.alpha || defaultAlpha);
          
          const outerRing = polygonCoords[0]; // O primeiro anel é sempre o exterior

          const pixiPoints = outerRing.map(c => {
            const projected = project(L.latLng(c[1], c[0]));
            return [projected.x, projected.y];
          }).flat();

          graphics.drawPolygon(pixiPoints);

          // Se houver buracos (anéis internos), desenhe-os também
          for (let i = 1; i < polygonCoords.length; i++) {
            const holeRing = polygonCoords[i];
            const holePixiPoints = holeRing.map(c => {
              const projected = project(L.latLng(c[1], c[0]));
              return [projected.x, projected.y];
            }).flat();
            graphics.drawPolygon(holePixiPoints);
          }
          graphics.endFill();

          graphics.interactive = false;
          graphics.buttonMode = false;
          if (!name) {
            graphics.interactive = true;
            graphics.buttonMode = true;
            graphics.on('pointerdown', () => {
              alert('Clicou no polígono: ' + feature.properties.name);
            });
          }

         container.addChild(graphics);
        });
      });
    };

    // --- Desenha as Camadas Padrão ---
    drawFeatures(geojsonNatalInteiroDefault.features);
    drawFeatures(geojsonPontosNatalDefault.features);

    // --- Desenha a Camada Dinâmica (recebida via prop) ---
    // A cor padrão para a camada dinâmica pode ser diferente para destaque
    drawFeatures(dynamicGeojsonLayer?._rawValue?.features, 0xFFD700, 0.7, 'dynGeoJson'); // Dourado com mais opacidade    

    renderer.render(container);
  }, pixiContainer);

  overlay.addTo(map);

  // Observa a prop 'dataGeojson' para atualizar a camada dinâmica
  watch(() => props.dataGeojson, (newVal) => {
    dynamicGeojsonLayer.value = newVal; // Atualiza o ref interno com o novo GeoJSON
    if (overlay) {
      overlay.redraw(); // Força o redesenho do overlay para exibir o novo GeoJSON
    }
  }, { immediate: true }); // 'immediate: true' faz com que o watcher seja executado imediatamente na montagem

  // 🟩 Exemplo: Troca a cor de um polígono na camada padrão "Pontos de Natal" depois de 5 segundos
  // Guarde o ID do intervalo
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
  if (map) map.remove();
  if (ticker) ticker.stop(); // Limpa o intervalo ao desmontar
});
</script>

<style>
#map {
  width: 100vw;
  height: 100vh;
  position: fixed;
  background-color: #f0f0f0;
}
html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  overflow: hidden;
}
</style>
