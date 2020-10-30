<template>
  <div class="home">
    <!-- Just a quick notice to the user that it can be interacted with -->
    <!-- <span class="drag-notice" id="js-drag-notice">Drag to rotate 360&#176;</span> -->
    <!-- These toggle the the different parts of the chair that can be edited, note data-option is the key that links to the name of the part in the 3D file -->
    <div class="options">
      <div
        class="option"
        :class="{ '--is-active': activeOption === 'legs' }"
        @click="selectOption('legs')"
      >
        <img
          src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/legs.svg"
          alt=""
        />
      </div>
      <div
        class="option"
        :class="{ '--is-active': activeOption === 'cushions' }"
        @click="selectOption('cushions')"
      >
        <img
          src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/cushions.svg"
          alt=""
        />
      </div>
      <div
        class="option"
        :class="{ '--is-active': activeOption === 'base' }"
        @click="selectOption('base')"
      >
        <img
          src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/base.svg"
          alt=""
        />
      </div>
      <div
        class="option"
        :class="{ '--is-active': activeOption === 'supports' }"
        @click="selectOption('supports')"
      >
        <img
          src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/supports.svg"
          alt=""
        />
      </div>
      <div
        class="option"
        :class="{ '--is-active': activeOption === 'back' }"
        @click="selectOption('back')"
      >
        <img
          src="https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/back.svg"
          alt=""
        />
      </div>
    </div>

    <!-- container -->
    <div id="container"></div>

    <!-- controls -->
    <div class="controls">
      <div class="info">
        <div class="info__message">
          <p>
            <strong>&nbsp;Grab&nbsp;</strong> to rotate chair.
            <strong>&nbsp;Scroll&nbsp;</strong> to zoom.
            <strong>&nbsp;Drag&nbsp;</strong> swatches to view more.
          </p>
        </div>
      </div>
      <!-- This tray will be filled with colors via JS, and the ability to slide this panel will be added in with a lightweight slider script (no dependency used for this) -->
      <div class="tray">
        <div class="tray__slide">
          <div
            class="tray__swatch"
            v-for="(color, index) in colors"
            :key="index"
            @click="selectSwatch(index)"
            :style="{
              background: color.texture
                ? 'url(' + color.texture + ')'
                : '#' + color.color
            }"
          ></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { ColorsData } from "@/static/index";

const DRAG_NOTICE = document.getElementById("js-drag-notice");
const BACKGROUND_COLOR = 0xf1f1f1;
const MODEL_PATH = //"../assets/materials/chair.glb";
  "https://s3-us-west-2.amazonaws.com/s.cdpn.io/1376484/chair.glb";
let theModel;
let initRotate = 0;

export default {
  name: "Home",
  data() {
    return {
      isLoading: true,
      camera: null,
      scene: null,
      renderer: null,
      controls: null,
      mesh: null,
      colors: ColorsData,
      activeOption: "legs"
    };
  },
  methods: {
    init: function() {
      // Init the scene
      this.scene = new THREE.Scene();
      // Set background
      this.scene.background = new THREE.Color(BACKGROUND_COLOR);
      this.scene.fog = new THREE.Fog(BACKGROUND_COLOR, 20, 100);

      // Init the renderer
      const container = document.querySelector("#container");
      this.renderer = new THREE.WebGLRenderer({ antialias: true });

      this.renderer.shadowMap.enabled = true;
      this.renderer.setPixelRatio(container.devicePixelRatio);
      container.appendChild(this.renderer.domElement);

      // Add a camerra
      this.camera = new THREE.PerspectiveCamera(
        50,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );
      this.camera.position.z = 5;
      this.camera.position.x = 0;

      // Initial material
      const INITIAL_MTL = new THREE.MeshPhongMaterial({
        color: 0xf1f1f1,
        shininess: 10
      });

      const INITIAL_MAP = [
        { childID: "back", mtl: INITIAL_MTL },
        { childID: "base", mtl: INITIAL_MTL },
        { childID: "cushions", mtl: INITIAL_MTL },
        { childID: "legs", mtl: INITIAL_MTL },
        { childID: "supports", mtl: INITIAL_MTL }
      ];

      // Init the object loader
      const loader = new GLTFLoader();

      loader.load(
        MODEL_PATH,
        gltf => {
          theModel = gltf.scene;

          theModel.traverse(o => {
            if (o.isMesh) {
              o.castShadow = true;
              o.receiveShadow = true;
            }
          });

          // Set the models initial scale
          theModel.scale.set(2, 2, 2);
          theModel.rotation.y = Math.PI;

          // Offset the y position a bit
          theModel.position.y = -1;

          // Set initial textures
          for (let object of INITIAL_MAP) {
            this.initColor(theModel, object.childID, object.mtl);
          }

          // Add the model to the scene
          this.scene.add(theModel);

          // Remove the loader
          // this.isLoading = false;
        },

        xhr => {
          // called while loading is progressing
          console.log(`${(xhr.loaded / xhr.total) * 100}% loaded`);
        },

        error => {
          console.error("error", error);
        }
      );

      // Add lights
      var hemiLight = new THREE.HemisphereLight(0xffffff, 0xffffff, 0.61);
      hemiLight.position.set(0, 50, 0);
      // Add hemisphere light to scene
      this.scene.add(hemiLight);

      var dirLight = new THREE.DirectionalLight(0xffffff, 0.54);
      dirLight.position.set(-8, 12, 8);
      dirLight.castShadow = true;
      dirLight.shadow.mapSize = new THREE.Vector2(1024, 1024);
      // Add directional Light to scene
      this.scene.add(dirLight);

      // Floor
      var floorGeometry = new THREE.PlaneGeometry(5000, 5000, 1, 1);
      var floorMaterial = new THREE.MeshPhongMaterial({
        color: 0xeeeeee,
        shininess: 0
      });

      var floor = new THREE.Mesh(floorGeometry, floorMaterial);
      floor.rotation.x = -0.5 * Math.PI;
      floor.receiveShadow = true;
      floor.position.y = -1;
      this.scene.add(floor);

      // Add controls
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.maxPolarAngle = Math.PI / 2;
      this.controls.minPolarAngle = Math.PI / 3;
      this.controls.enableDamping = true;
      this.controls.enablePan = false;
      this.controls.dampingFactor = 0.1;
      this.controls.autoRotate = false; // Toggle this if you'd like the chair to automatically rotate
      this.controls.autoRotateSpeed = 0.2; // 30
    },

    // Function - Add the textures to the models
    initColor(parent, type, mtl) {
      parent.traverse(o => {
        if (o.isMesh) {
          if (o.name.includes(type)) {
            o.material = mtl;
            o.nameID = type; // Set a new property to identify this object
          }
        }
      });
    },

    animate() {
      this.controls.update();
      this.renderer.render(this.scene, this.camera);
      requestAnimationFrame(this.animate);

      if (this.resizeRendererToDisplaySize(this.renderer)) {
        const canvas = this.renderer.domElement;
        this.camera.aspect = canvas.clientWidth / canvas.clientHeight;
        this.camera.updateProjectionMatrix();
      }

      if (theModel !== null && this.isLoading === false) {
        this.initialRotation();
        DRAG_NOTICE.classList.add("start");
      }
    },

    initialRotation() {
      initRotate++;
      if (initRotate <= 120) {
        theModel.rotation.y += Math.PI / 60;
      } else {
        this.isLoading = true;
      }
    },

    resizeRendererToDisplaySize(renderer) {
      const canvas = renderer.domElement;
      var width = window.innerWidth;
      var height = window.innerHeight;
      var canvasPixelWidth = canvas.width / window.devicePixelRatio;
      var canvasPixelHeight = canvas.height / window.devicePixelRatio;

      const needResize =
        canvasPixelWidth !== width || canvasPixelHeight !== height;
      if (needResize) {
        renderer.setSize(width, height, false);
      }
      return needResize;
    },

    // select options
    selectOption(data) {
      this.activeOption = data;
    },

    // select swatch
    selectSwatch(index) {
      let color = ColorsData[index];
      let new_mtl;
      if (color.texture) {
        let txt = new THREE.TextureLoader().load(color.texture);

        txt.repeat.set(color.size[0], color.size[1], color.size[2]);
        txt.wrapS = THREE.RepeatWrapping;
        txt.wrapT = THREE.RepeatWrapping;

        new_mtl = new THREE.MeshPhongMaterial({
          map: txt,
          shininess: color.shininess ? color.shininess : 10
        });
      } else {
        new_mtl = new THREE.MeshPhongMaterial({
          color: parseInt("0x" + color.color),
          shininess: color.shininess ? color.shininess : 10
        });
      }
      this.setMaterial(theModel, this.activeOption, new_mtl);
    },

    setMaterial(parent, type, mtl) {
      parent.traverse(o => {
        if (o.isMesh && o.nameID != null) {
          if (o.nameID == type) {
            o.material = mtl;
          }
        }
      });
    }
  },
  mounted() {
    this.init();
    this.animate();
  }
};
</script>
