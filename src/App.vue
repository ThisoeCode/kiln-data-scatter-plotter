<template>
  <main>
    <section>
      <label for="xls">上传Excel文件</label>
      <input id="xls" type="file" accept=".xlsx, .xls" @change="handleFileUpload" />
    </section>
    <div id="three-container"></div>
  </main>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import * as XLSX from 'xlsx'

// YaoKou colors
const specificKilnColors = {
  'yin gou ruins': 0x123456,
  'yao zhou kiln': 0x654321,
};

export default {

name: 'ThreeDScatterPlot',

data() {
  return {
    renderer: null,
    camera: null,
    controls: null,
    data: [], // storage of kiln-data from xls
    cameraAdjusted: false, // 添加一个标志，以确定是否已经调整过相机 ???????
    bounds: null // 用于存储数据点边界 ???????
  };
},

mounted() {
  this.initThree();
  window.addEventListener('resize', this.onWindowResize);
  this.animate = this.animate.bind(this);
  this.adjustCamera = this.adjustCamera.bind(this);

  this.animate();
},

beforeUnmount() {
  if (this.controls) this.controls.dispose();
  window.removeEventListener('resize', this.onWindowResize);
  if (this.renderer) this.renderer.dispose();
},

/* ------- MAIN ------- */
methods: {

handleFileUpload(event) {
  const file = event.target.files[0];
  if (!file) { return }
  const reader = new FileReader();
  reader.onload = (e) => {
    // Read xls
    const xls = XLSX.read(e.target.result, { type: 'binary' }); 
    this.data = this.mapXLS( XLSX.utils.sheet_to_json(xls.Sheets[xls.SheetNames[0]]) );

    // plot
    this.createPlot();
    this.adjustCamera();
  };
  reader.readAsBinaryString(file);
},

mapXLS(main_data) {
  return main_data.map(item => {
    return {
      kiln: item.kiln,
      Na2O: parseFloat(item.Na2O),
      MgO: parseFloat(item.MgO),
      Al2O3: parseFloat(item.Al2O3),
      SiO2: parseFloat(item.SiO2),
      K2O: parseFloat(item.K2O),
      CaO: parseFloat(item.CaO),
      TiO2: parseFloat(item.TiO2),
      Fe2O3: parseFloat(item.Fe2O3)
    };
  });
},

createPlot() {
  // calc coordinates
  const dataObj = this.data.map(item => {
    const kilnKey = item.kiln.toLowerCase();
    /** @summary Specific Kiln Color @description Set a random color when is some specific kiln */
    const SKC = specificKilnColors[kilnKey] || Math.random() * 0xffffff;
    return {
      kiln: item.kiln,
      x: item.Na2O + item.MgO + item.K2O + item.CaO , // alkaline oxides sum
      y: item.Al2O3 + item.Fe2O3 , // neutral oxide sum
      z: item.SiO2 + item.TiO2 , // acidic oxides sum
      color: SKC
    };
  });

  /* ------- CONFIG THREE ------- */
  const geometry = new THREE.BufferGeometry();
  const positions = [];
  const colors = [];
  dataObj.forEach(item => {
    positions.push(item.x, item.y, item.z);
    const color3 = new THREE.Color(item.color);
    colors.push(color3.r, color3.g, color3.b);
  });

  geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions,3));
  geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));

  const material = new THREE.PointsMaterial({ size: 0.5, vertexColors: true });
  const points = new THREE.Points(geometry, material);
  this.scene.add(points);

  // Ctrl+F 'this.bounds' , NOT WORKING ???????
  this.calculateBounds();
  
  // 添加坐标轴标签 ???????
  this.addAxesLabels();
  
  // !!!!!!! 在这里我们调用一个新的函数来调整相机
  this.adjustCamera();
},

calculateBounds() {
  const bounds = {
    minX: Infinity,
    maxX: -Infinity,
    minY: Infinity,
    maxY: -Infinity,
    minZ: Infinity,
    maxZ: -Infinity
  };
  
  this.data.forEach(item => {
    bounds.minX = Math.min(bounds.minX, item.x);
    bounds.maxX = Math.max(bounds.maxX, item.x);
    bounds.minY = Math.min(bounds.minY, item.y);
    bounds.maxY = Math.max(bounds.maxY, item.y);
    bounds.minZ = Math.min(bounds.minZ, item.z);
    bounds.maxZ = Math.max(bounds.maxZ, item.z);
  });
  
  this.bounds = bounds;
  // console.log(bounds) // => ALL VALUE IS "NaN"
},

addAxesLabels() {
  const axesHelper = new THREE.AxesHelper(5);
  this.scene.add(axesHelper);

  const createAxisLabel = (text, position, color) => {
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    context.font = '24px Arial';
    context.fillStyle = color;
    context.fillText(text, 10, 50);

    const texture = new THREE.Texture(canvas);
    texture.needsUpdate = true;

    const material = new THREE.SpriteMaterial({ map: texture });
    const sprite = new THREE.Sprite(material);
    sprite.position.copy(position);
    sprite.scale.set(0.5, 0.25, 1);
    this.scene.add(sprite);
  };

  createAxisLabel('X', new THREE.Vector3(5.5, 0, 0), 'red');
  createAxisLabel('Y', new THREE.Vector3(0, 5.5, 0), 'green');
  createAxisLabel('Z', new THREE.Vector3(0, 0, 5.5), 'blue');
},
    
adjustCamera() {
  const boundingBox = new THREE.Box3();

  this.data.forEach(item => {
    boundingBox.expandByPoint(new THREE.Vector3(item.x, item.y, item.z));
  });

  const center = boundingBox.getCenter(new THREE.Vector3());
  const size = boundingBox.getSize(new THREE.Vector3());
  const maxDim = Math.max(size.x, size.y, size.z);
  const fov = this.camera.fov * (Math.PI / 180);

  // making sure that  maxDim != 0
  let cameraZ = maxDim > 0 ? Math.abs(maxDim / 2 / Math.tan(fov / 2)) : 10; 
  cameraZ *= 1.5; // modify cam perspective

  this.camera.position.set(center.x, center.y, cameraZ);
  this.camera.lookAt(center);
  this.camera.updateProjectionMatrix();
  this.controls.target.copy(center);

  this.renderer.render(this.scene, this.camera);

  // console.log("center of bound-box：", center);
  // console.log("camera-z：", cameraZ);
  // console.log("cam position：", this.camera.position);
},
  
initThree() {
  // init THREE
  const container = document.getElementById('three-container');

  // init cam
  this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  this.camera.position.z = 15;

  // init WebGLRenderer
  this.renderer = new THREE.WebGLRenderer({ antialias: true });
  this.renderer.setClearColor(0xffffff, 1); // background color
  this.renderer.setSize(window.innerWidth, window.innerHeight);
  container.appendChild(this.renderer.domElement);

  this.scene = new THREE.Scene();


  // plot square grid
  const size = 10;
  const divisions = 10;
  this.scene.add( new THREE.GridHelper(size, divisions) );
  // plot axes
  const axesHelper = new THREE.AxesHelper(5);
  this.scene.add(axesHelper);

  // init controller
  this.controls = new OrbitControls(this.camera, this.renderer.domElement);

  // loop
  this.animate();
},

animate() {
  requestAnimationFrame(this.animate);

  // update controller ONLY IF cam not adjusted  !*** USED ONLY, BUT NOT BEEN MODDED ***!
  if (!this.cameraAdjusted) {
    this.controls.update();
  }

  this.renderer.render(this.scene, this.camera);
},

onWindowResize() {
  this.camera.aspect = window.innerWidth / window.innerHeight;
  this.camera.updateProjectionMatrix();
  this.renderer.setSize(window.innerWidth, window.innerHeight);
},



  }, // vue - methods
};
</script>

<style>
body { margin:0; padding:0; }
main{
  display:flex; flex-direction:column;
  width:100vw; height:100vh;
  overflow:hidden;
}
section{
  display:flex; flex-direction:column;
  margin: auto 9pt;
  align-items: center;
  user-select:none;
}
label{
  line-height:2;
  font-size:16pt;font-weight:bold;
}
#xls{
  padding:7pt;
  border:thin gray dotted;
  border-radius:9px;
  background-color:#ccc;
  margin-bottom:7px;
}
#three-container{
  margin:auto 0; padding:0;
  width:100%; height: auto;
  border-top:3px solid gray;
}
</style>