<template>
  <div id="app">
    
    <header>
      <div class="header-left">
        <div class="logo">
          <img alt="logo" src="./assets/fire-logo.png" />
        </div>
        <h3>{{currentCity}} Burn Severity</h3>
      </div>
      <div class="header-right">
        <button @click="zoomToHouse">View Property</button>
      </div>
    </header>

    <main>
      <div id="cesiumContainer"></div>
      <div class="controls">
        <div class="controls-title">Selection</div>
        <select v-model="selected">
          <option v-for="fire in fireData" :value="fire.name" :key="fire.name">
            {{ fire.name }}
          </option>
        </select>
        <div class="controls-title">Active</div>
        <div class="activeData" v-if="activeFire">
          <div class="title">{{ activeFire.name}}</div>
          <div class="description">{{ activeFire.description[0] }}</div> 
          <div class="description">{{ activeFire.description[1] }}</div> 
        </div>
      </div>
    </main>

  </div>
</template>

<script>
import {
  Cartesian3,
  Math as CesiumMath,
  Viewer,
  Ion,
  IonResource,
  KmlDataSource,
  IonImageryProvider,
  createOsmBuildingsAsync,
  createWorldTerrainAsync,
  ImageMaterialProperty,
  Cesium3DTileset,
  Color,
} from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";

//non reactive data reference
let data = [];
let house;

export default {
  name: 'App',
  data() {
    return {
      viewer: null,
      currentCity: 'Oakridge', // TODO: make selection that drives data pull
      fireTileSets: [2516132, 2517505, 2517511, 2517527, 2517530], // TODO: pull dynamically using tags
      selected: null,
      fireData: [], // type: { name: string, description: string, show: bool, tileset: number, dataSource: any }
      token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI1NjY4NDM5MC03YjQ5LTQyMGUtYTlmZC02M2Q3ZGVlYmRlYjEiLCJpZCI6MTcwODg5LCJpYXQiOjE3MTEyOTk5OTh9.ST3QNmGexguGSKpUtV_wvNsatQXvFanItK5AGxHa7PE'
    }
  },
  computed: {
    activeFire: {
      get() {      
        return this.fireData.find(fire => fire.name === this.selected);
      }
    }
  },
  watch: {
    activeFire () {
      let refData = data.find(fire => fire.name === this.activeFire.name);
      this.viewer.zoomTo(refData.dataSource);
    }
  },
  methods: {

    zoomToHouse() {
      if (!house) return;
      this.viewer.zoomTo(house);
    },

    generateFireData() {
      this.fireTileSets.forEach(async (tileset) => {

        let fireData = {};
        const resource = await IonResource.fromAssetId(tileset);
        const dataSource = await KmlDataSource.load(resource, {
          camera: this.viewer.scene.camera,
          canvas: this.viewer.scene.canvas,
        });
        //hide unneeded visuals from dataset
        var entities = dataSource.entities.values;
        entities.forEach(entity => {
          if (entity.name === "Pre-Fire Landsat Image" || entity.name === "Post-Fire Landsat Image") {
            entity.show = false; // Toggle visibility based on the parameter
          }
          if (entity.rectangle) {
            let image = entity.rectangle.material.image._value
            entity.rectangle.material = new ImageMaterialProperty({
              transparent: true,
              image: image,
              color: Color.WHITE.withAlpha(0.35),
            });
          }
        });
        //get metadata
        await fetch(`https://api.cesium.com/v1/assets/${tileset}?access_token=${Ion.defaultAccessToken}`)
          .then(response => response.json())
          .then(data => {
            fireData.name = data.name;
            fireData.description = ['',''];
            let description = data.description.split(',')
            fireData.description[0] = description[0] ? description[0] : '';
            fireData.description[1] = description[1] ? description[1] : '';
          })
          .catch(error => { console.error('Error:', error); });
        
        //set properties
        fireData.show = true;
        fireData.tileset = tileset;
        fireData.dataSource = dataSource;
        //push reactive reference
        this.fireData.push(fireData);
        //push non-reactive reference (used for zoomTo method)
        data.push(fireData);
        //set UI
        this.selected = this.fireData[0].name;
        //add to viewer
        await this.viewer.dataSources.add(dataSource);
      });
    },
  },

  async mounted() {

    // Set the access token
    Ion.defaultAccessToken = this.token;

    // initialize the viewer without default ui
    this.viewer = new Viewer("cesiumContainer", {
      //baseLayerPicker: false,      
      //navigationHelpButton: false, 
      animation: false,
      fullscreenButton: false,
      vrButton: false,
      geocoder: false,
      homeButton: false,
      infoBox: false,
      sceneModePicker: false,
      selectionIndicator: false,
      timeline: false,
      navigationInstructionsInitiallyVisible: false,
    });

    // allow depth testing against terrain
    const scene = this.viewer.scene;
    scene.globe.depthTestAgainstTerrain = true;

    // Add Cesium World Terrain, a global terrain layer.
    createWorldTerrainAsync().then((worldTerrain) => {
      this.viewer.scene.terrainProvider = worldTerrain;
      scene.globe.show = true;
    });

    // Add Cesium OSM Buildings, a global 3D buildings layer.
    createOsmBuildingsAsync().then((buildingTileset) => {
      this.viewer.scene.primitives.add(buildingTileset);
      this.tileset = buildingTileset;
    });

    // Add Bing Maps Labels Only to the right panel
    this.viewer.imageryLayers.addImageryProvider(
      await IonImageryProvider.fromAssetId(2411391),
    );

    // Add home gltf model for scale / reference
    house = this.viewer.scene.primitives.add(
      await Cesium3DTileset.fromIonAssetId(2517636),
    );

    //load fire data
    this.generateFireData();
  }
}
</script>

<style scoped>

header {
  height: 55px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px;
  border-bottom: 1px solid #27272a;
}

.header-left {
  display: flex;
  align-items: center;
}

.logo {
  width: 35px;
  height: 35px;
  border-radius: 100%;
  margin-right: 20px;
  border: 1px solid #27272a;
  border-radius: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 5px;
}
.logo img {
  width: 100%;
  height: 100%;
}

main {
  height: calc(100vh - 55px);
  width: 100vw;
  padding: 20px;
  display: flex;
  flex-direction: row;
}

#cesiumContainer {
  height: 100%;
  width: 100%;
  border-radius: 10px;
  overflow: hidden;
}

.controls {
  width: 350px;
  padding: 15px 15px;
  margin-left: 20px;
  border: 1px solid #27272a;
  border-radius: 10px;
}
.controls-title {
  font-size: 13px;
  margin-top: 40px;
}
.controls-title:first-of-type {
  margin-top: 0;
}

.title {
  font-size: 22px;
  font-weight: bold;
  margin: 5px 0;
}
.description {
  margin-top: 3px;
}

</style>
