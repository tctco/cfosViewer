<template>
  <canvas id="renderCanvas"></canvas>
  <div class="slidecontainer">
    <label>Point Cloud Size:
      <input type="range" min="0" max="10" class="slider" v-model="this.pcsSize" @change="handlePcsSizeChange">
    </label>
    <label>Model Visibility:
      <input type="range" min="0" max="1" step="0.1" class="slider" v-model="this.modelVisibility"
        @change="handleModelVisibilityChange">
    </label>
    <label>Glow Intensity:
      <input type="range" min="0" max="1" step="0.1" class="slider" v-model="this.glowIntensity"
        @change="handleGlowIntensityChange">
    </label>
    <label>Background Brightness:
      <input type="range" min="0.7" max="1" step="0.03" class="slider" v-model="this.brightness"
        @change="handleBrightnessChange">
    </label>
    <label>Color:
      <color-picker v-model:pureColor="this.color" v-on:pure-color-change="handleColorChange" />
    </label>
    <label>Display AxesViewer:
      <input type="checkbox" v-model="displayAxes" @change="handleAxesToggleChange"/>
    </label>

  </div>
</template>

<script>
import * as BABYLON from "babylonjs"
import "babylonjs-loaders"
import axios from 'axios'
import { ColorPicker } from 'vue3-colorpicker'
import "vue3-colorpicker/style.css"

function parseColor(input) {
  return input.split("(")[1].split(")")[0].split(",").map(parseFloat)
}

export default {
  components: { ColorPicker },
  data() {
    return {
      pcsSize: 2,
      axis: null,
      scene: null,
      camera: null,
      pcs: null,
      model: null,
      modelVisibility: .5,
      gl: null,
      glowIntensity: .2,
      color: 'rgba(204,230,20,0.9)',
      brightness: .9,
      displayAxes: true
    }
  },
  methods: {
    handlePcsSizeChange: function () {
      this.pcs.mesh.material.pointSize = this.pcsSize
    },
    handleModelVisibilityChange: function () {
      this.model.visibility = this.modelVisibility
    },
    handleGlowIntensityChange: function () {
      this.gl.intensity = this.glowIntensity
    },
    handleBrightnessChange: function () {
      this.scene.clearColor = new BABYLON.Color3(this.brightness, this.brightness, this.brightness)
    },
    handleColorChange: function (color) {
      if (!color || !this.pcs) return
      let rgba = parseColor(color)
      if (rgba.length === 3) rgba.push(1)
      for (let i = 0; i < 3; i++) rgba[i] = rgba[i] / 255
      let shouldUpdateColor = new BABYLON.Color4(...rgba)
      for (let p = 0; p < this.pcs.nbParticles; p++) {
        this.pcs.particles[p].color = shouldUpdateColor
      }
      this.pcs.setParticles()
    },
    handleAxesToggleChange: function() {
      if (this.displayAxes) this.axes = new BABYLON.AxesViewer(this.scene, 1)
      else this.axes.dispose()
    },
    handleButtonClick: function () { console.log(typeof this.color) }
  },
  mounted() {
    const canvas = document.getElementById("renderCanvas")
    const engine = new BABYLON.Engine(canvas, true)
    this.scene = new BABYLON.Scene(engine);
    this.camera = new BABYLON.ArcRotateCamera("Camera", Math.PI / 2, Math.PI / 2, 2, new BABYLON.Vector3(0, 0, 5), this.scene);
    this.camera.attachControl(canvas, true);
    this.scene.clearColor = new BABYLON.Color3(.9, .9, .9);

    new BABYLON.HemisphericLight("light1", new BABYLON.Vector3(0, 0, 1), this.scene, scene => console.log(scene));
    this.camera.wheelPrecision = 40
    this.axes = new BABYLON.AxesViewer(this.scene, 1)

    let importPromise = BABYLON.SceneLoader.ImportMeshAsync("", '/src/assets/', 'recreated.glb', this.scene, mesh => console.log(mesh))
    importPromise.then((result) => {
      this.model = result.meshes[1];
      this.camera.setTarget(result.meshes[0])
      this.model.visibility = this.modelVisibility
    })
    axios.get('/src/assets/dots.json').then(res => {
      let dots = res.data.dots
      this.pcs = new BABYLON.PointsCloudSystem("pcs", this.pcsSize, this.scene)
      this.pcs.addPoints(dots.length, (p, i, s) => {
        p.position = new BABYLON.Vector3(-dots[i][0] - 0.8, dots[i][1] - 0.3, dots[i][2] + 0.1)
        p.color = new BABYLON.Color4(.8, .9, .08, .8)
      })
      this.pcs.buildMeshAsync().then(res => {
        this.gl = new BABYLON.GlowLayer('glow', this.scene)
        this.gl.addIncludedOnlyMesh(res)
        this.gl.referenceMeshToUseItsOwnMaterial(res);
        this.gl.intensity = this.glowIntensity
      })
    })

    engine.runRenderLoop(() => {
      this.scene.render();
    });

    window.addEventListener("resize", function () {
      engine.resize();
    });
  }
}

</script>

<style>
#renderCanvas {
  width: 100%;
  height: 100%;
}

label {
  margin: 10px;
}
</style>