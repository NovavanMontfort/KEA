<template>
  <div>
    <!-- Just an empty section to provide scroll space above the 3D content -->
    <section style="height: 100vh;"></section>

    <!-- This is where the 3D scene is rendered -->
    <section class="three-container" ref="threeContainer"></section>

    <!-- Extra scroll space below the 3D content -->
    <section style="height: 2000px;"></section>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

// Reference to the container where the 3D scene will be rendered
const threeContainer = ref(null)

// Variables for Three.js scene, camera, renderer, and animation loop
let scene, camera, renderer, animationId

// 3D model (will be loaded dynamically)
let model = null

// Used to throttle scroll event
let scrollTimeout = null
let isScrolling = false

// Target values for position, rotation, and scale (used for smooth animation)
let targetX = 0
let targetRotationY = 0
let targetRotationZ = 0     // EXTRA - "tumbling" animation
let targetScale = 1

// Used for vertical floating animation
let floatTime = 0;    // EXTRA - floating animation

// Constants to control model visibility limits on the X-axis
const visibleLimit = 4.5
const outOfViewOffset = 1.5

onMounted(() => {
  // Create scene and set background color
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#F8F8FF')

  // ✅ Eerst scene aanmaken
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#F8F8FF')

  // Set up camera with perspective projection
  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5

  // Set up renderer with anti-aliasing and size
  renderer = new THREE.WebGLRenderer({ antialias: true })

  //Activates shadows
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  // Add renderer canvas to the container
  threeContainer.value.appendChild(renderer.domElement)

  // Load a 3D model (GLB format)
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', (gltf) => {
    model = gltf.scene
    scene.add(model)

  //lighting for shadows
    model.traverse((child) => {
  if (child.isMesh) {
    child.castShadow = true
    child.receiveShadow = true
    child.material.flatShading = true  // optioneel, voor duidelijke vlakken
  }
  })

    model.position.set(0, 0, 0)

    model.scale.set(1, 1, 1) 

  })

  // Ambient light - verlaag intensiteit voor meer schaduwcontrast
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3)
  scene.add(ambientLight)

  // Directional light met schaduw
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1.2)
  directionalLight.position.set(5, 10, 7)
  directionalLight.castShadow = true

  // Shadow instellingen voor betere kwaliteit
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  directionalLight.shadow.camera.near = 0.5
  directionalLight.shadow.camera.far = 50
  directionalLight.shadow.camera.left = -10
  directionalLight.shadow.camera.right = 10
  directionalLight.shadow.camera.top = 10
  directionalLight.shadow.camera.bottom = -10

  scene.add(directionalLight)

  // Extra fill light om donkere kanten wat op te lichten
  const fillLight = new THREE.PointLight(0xffffff, 0.4, 50)
  fillLight.position.set(-5, 5, -5)
  scene.add(fillLight)


  // Start animation loop
  animate()

  // Set up event listeners
  window.addEventListener('resize', onWindowResize)
  window.addEventListener('scroll', handleScroll)
})

onBeforeUnmount(() => {
  // Clean up event listeners and stop animation when component is removed
  window.removeEventListener('resize', onWindowResize)
  window.removeEventListener('scroll', handleScroll)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})

// Handle browser window resize: update camera and renderer size
function onWindowResize() {
  if (!threeContainer.value) return

  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

// Animation loop
function animate() {
  animationId = requestAnimationFrame(animate)

  if (model) {
    // position on the X-axis is smoothly interpolated
    model.position.x += (targetX - model.position.x) * 0.1

    // Y-rotation stays as you set it before
    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07

    // EXTRA - "tumbling" animation
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07
 
    // Scale interpolatie
    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    // EXTRA - floating animation
    floatTime += 0.02; // change speed of the movement by increasing or decreasing the value
    model.position.y = Math.sin(floatTime) * 0.1; // 0.1 is the sizing of the floating
  }

  // Renders the scene from the perspective of the camera
  renderer.render(scene, camera)
}

// Handle scroll interaction: update animation targets based on scroll progress
function handleScroll() {
  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))

  // Throttle scroll updates
  isScrolling = true
  if (scrollTimeout !== null) clearTimeout(scrollTimeout)
  scrollTimeout = setTimeout(() => (isScrolling = false), 100)

  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  // Move from center to right, then left based on scroll
  if (scrollProgress <= 0.5) {
    const t = scrollProgress * 2
    targetX = THREE.MathUtils.lerp(0, rightEndX, t)
  } else {
    const t = (scrollProgress - 0.5) * 2
    targetX = THREE.MathUtils.lerp(rightEndX, leftEndX, t)
  }

  // Y-rotatie: back and forth symmetrical
  targetRotationY = Math.sin(scrollProgress * Math.PI) * (Math.PI / 2)

  // Z-rotatie: on scroll
  targetRotationZ = scrollProgress * Math.PI * 2

  // Scale
  targetScale = THREE.MathUtils.lerp(1, 2.5, scrollProgress)
}
</script>

<style scoped>
/* Full height for base elements */
html, body, #__nuxt, #app {
  height: 100%;
  margin: 0;
  padding: 0;
}

/* Container for the Three.js canvas */
.three-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none; /* Makes sure the 3D scene doesn't block interaction with page content */
  z-index: 10;
}
</style>





