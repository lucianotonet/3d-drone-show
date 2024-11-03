<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { gsap } from 'gsap';
import DroneControls from './DroneControls.vue';

const canvas = ref<HTMLCanvasElement | null>(null);
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let drones: Drone[] = [];

const initialCameraPosition = {
  position: new THREE.Vector3(0, 20, 50),
  target: new THREE.Vector3(0, 0, 0)
};

interface DronePosition {
  x: number;
  y: number;
  z: number;
  color?: string;
}

class Drone {
  mesh: THREE.Mesh;
  targetPosition: THREE.Vector3;
  initialPosition: THREE.Vector3;

  constructor(position: THREE.Vector3, color: string = '#ffffff') {
    const geometry = new THREE.SphereGeometry(0.1, 8, 8);
    const material = new THREE.MeshBasicMaterial({
      color: new THREE.Color(color),
      transparent: true,
      opacity: 0.8,
    });
    
    this.mesh = new THREE.Mesh(geometry, material);
    this.mesh.position.copy(position);
    this.targetPosition = position.clone();
    this.initialPosition = position.clone();
  }

  updatePosition(delta: number) {
    this.mesh.position.lerp(this.targetPosition, delta * 2);
  }

  setTarget(position: THREE.Vector3, duration: number = 1) {
    this.targetPosition.copy(position);
    gsap.to(this.mesh.position, {
      x: position.x,
      y: position.y,
      z: position.z,
      duration,
      ease: "power2.inOut"
    });
  }

  setColor(color: string) {
    (this.mesh.material as THREE.MeshBasicMaterial).color.set(color);
  }
}

const createSkybox = () => {
  const geometry = new THREE.SphereGeometry(500, 60, 40);
  geometry.scale(-1, 1, 1);

  const uniforms = {
    topColor: { value: new THREE.Color(0x0077ff) },
    bottomColor: { value: new THREE.Color(0x000000) },
    offset: { value: 33 },
    exponent: { value: 0.6 }
  };

  const vertexShader = `
    varying vec3 vWorldPosition;
    void main() {
      vec4 worldPosition = modelMatrix * vec4(position, 1.0);
      vWorldPosition = worldPosition.xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `;

  const fragmentShader = `
    uniform vec3 topColor;
    uniform vec3 bottomColor;
    uniform float offset;
    uniform float exponent;
    varying vec3 vWorldPosition;
    void main() {
      float h = normalize(vWorldPosition + offset).y;
      gl_FragColor = vec4(mix(bottomColor, topColor, max(pow(max(h, 0.0), exponent), 0.0)), 1.0);
    }
  `;

  const material = new THREE.ShaderMaterial({
    uniforms: uniforms,
    vertexShader: vertexShader,
    fragmentShader: fragmentShader,
    side: THREE.BackSide
  });

  const sky = new THREE.Mesh(geometry, material);
  scene.add(sky);
};

const initScene = () => {
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x000000, 0.001);

  camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  camera.position.copy(initialCameraPosition.position);
  camera.lookAt(initialCameraPosition.target);

  renderer = new THREE.WebGLRenderer({
    canvas: canvas.value!,
    antialias: true,
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

  // Initialize OrbitControls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.maxDistance = 100;
  controls.minDistance = 5;

  // Add skybox
  createSkybox();

  // Create initial drone formation
  const droneCount = 8000;
  const gridSize = Math.ceil(Math.sqrt(droneCount));
  const spacing = 0.5;
  const offset = (gridSize * spacing) / 2;

  for (let i = 0; i < droneCount; i++) {
    const x = (i % gridSize) * spacing - offset;
    const y = Math.floor(i / gridSize) * spacing - offset;
    const z = 0;
    
    const drone = new Drone(
      new THREE.Vector3(x, y, z),
      `hsl(${(i / droneCount) * 360}, 70%, 50%)`
    );
    drones.push(drone);
    scene.add(drone.mesh);
  }

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  animate();
};

const animate = () => {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
};

// API for controlling drones and camera
const droneAPI = {
  // Camera controls
  resetCamera(duration: number = 2) {
    gsap.to(camera.position, {
      x: initialCameraPosition.position.x,
      y: initialCameraPosition.position.y,
      z: initialCameraPosition.position.z,
      duration,
      ease: "power2.inOut",
      onUpdate: () => {
        camera.lookAt(initialCameraPosition.target);
      }
    });
    
    gsap.to(controls.target, {
      x: initialCameraPosition.target.x,
      y: initialCameraPosition.target.y,
      z: initialCameraPosition.target.z,
      duration,
      ease: "power2.inOut"
    });
  },

  resetFormation(duration: number = 2) {
    drones.forEach(drone => {
      drone.setTarget(drone.initialPosition, duration);
    });
  },

  formSphere(radius: number = 10, duration: number = 2) {
    const phi = Math.PI * (3 - Math.sqrt(5));
    
    drones.forEach((drone, i) => {
      const y = 1 - (i / (drones.length - 1)) * 2;
      const r = Math.sqrt(1 - y * y);
      const theta = phi * i;
      
      const x = Math.cos(theta) * r * radius;
      const z = Math.sin(theta) * r * radius;
      
      drone.setTarget(new THREE.Vector3(x, y * radius, z), duration);
    });
  },

  formDoubleHelix(radius: number = 10, height: number = 20, duration: number = 2) {
    const turns = 3;
    drones.forEach((drone, i) => {
      const angle = (i / drones.length) * Math.PI * 2 * turns;
      const y = (i / drones.length) * height - height / 2;
      
      const x1 = Math.cos(angle) * radius;
      const z1 = Math.sin(angle) * radius;
      
      drone.setTarget(new THREE.Vector3(x1, y, z1), duration);
    });
  },

  // Reset everything (camera and formation)
  resetAll(duration: number = 2) {
    this.resetCamera(duration);
    this.resetFormation(duration);
  }
};

const handleResize = () => {
  if (camera && renderer) {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
};

onMounted(() => {
  initScene();
  window.addEventListener('resize', handleResize);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize);
});

defineExpose(droneAPI);
</script>

<template>
  <canvas ref="canvas" />
  <DroneControls
    :onReset="() => droneAPI.resetAll(2)"
    :onSphere="() => droneAPI.formSphere(10, 2)"
    :onHelix="() => droneAPI.formDoubleHelix(10, 20, 2)"
  />
</template>

<style scoped>
canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #000;
}
</style>