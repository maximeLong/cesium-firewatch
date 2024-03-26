# Cesium.js for Oregon Firewatch

See the live app [here](https://cesium-firewatch.vercel.app/).

## Project info

This project is made as demo to showcase some of the potential applications Cesium could provide in visualizing historical fire data on accurate 3D terrain. 

This project was inspired by work done by [HOWL](https://cesium.com/blog/2017/06/30/oregon-wildlands/) using the Cesium platform.

Fire data is taken from Monitoring Trends in Burn Severity [MTBS](https://mtbs.gov/), an interagency program who provides burn severity data across the United States.

### Cesium setup

Cesium requires some additional steps for Vue/Vite:
[Cesium.js and Vite](https://github.com/CesiumGS/cesium-vite-example)


### Project Setup and Hot-Reload

```sh
npm install
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```