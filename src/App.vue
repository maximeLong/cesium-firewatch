<template>
  <header>
    <h3>Wildfire Viewer</h3>
  </header>

  <main>
    <div id="cesiumContainer"></div>
  </main>
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
  Color,
} from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";

export default {
  name: 'App',
  data() {
    return {
      viewer: null,
    }
  },
  methods: {
    home() {
      this.viewer.camera.flyTo({
        destination: Cartesian3.fromDegrees(-122.4617, 43.7465, 25000),
        orientation: {
          heading: CesiumMath.toRadians(0.0),
          pitch: CesiumMath.toRadians(-45.0),
        },
      });
    },
    updateview(mode) {
           
    },
  },
  async mounted() {

    // Set the access token
    Ion.defaultAccessToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI1NjY4NDM5MC03YjQ5LTQyMGUtYTlmZC02M2Q3ZGVlYmRlYjEiLCJpZCI6MTcwODg5LCJpYXQiOjE3MTEyOTk5OTh9.ST3QNmGexguGSKpUtV_wvNsatQXvFanItK5AGxHa7PE';

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

    //load fire data
    const resource = await IonResource.fromAssetId(2516132);
    const dataSource = await KmlDataSource.load(resource, {
      camera: this.viewer.scene.camera,
      canvas: this.viewer.scene.canvas,
    });

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

    await this.viewer.dataSources.add(dataSource);

    // Fly to first fire
    // this.viewer.camera.flyTo({
    //   destination: Cartesian3.fromDegrees(-122.4617, 43.7465, 25000),
    //   orientation: {
    //     heading: CesiumMath.toRadians(0.0),
    //     pitch: CesiumMath.toRadians(-45.0),
    //   },
    // });
    
    await this.viewer.zoomTo(dataSource);


    //Construct the URL for the Cesium Ion API
    const url = `https://api.cesium.com/v1/assets/${2516132}?access_token=${Ion.defaultAccessToken}`;
    // Fetch the asset metadata
    fetch(url)
      .then(response => response.json())
      .then(data => {
        console.log('Asset Name:', data.name);
        console.log('Asset Description:', data.description);
        // Here, you can update your UI or take other actions with the asset's name and description
      })
      .catch(error => {
        console.error('Error:', error);
      });
  }
}
</script>

<style scoped>
header {
  height: 55px;
}

main {
  height: calc(100vh - 55px);
  width: 100vw;
  }

#cesiumContainer {
  height: 100%;
  width: 100%;
}

</style>
