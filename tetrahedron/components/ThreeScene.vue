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

const threeContainer = ref(null)

let scene, camera, renderer, animationId
let model = null

// Float animation
let floatTime = 0

// Flags & targets
const shouldRotate = ref(false)
let targetScale = 1

onMounted(() => {
  // Scene
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#F8F8FF')

  // Camera
  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

  // Load 3D model
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', (gltf) => {
    model = gltf.scene
    scene.add(model)

    model.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true
        child.receiveShadow = true
        child.material.flatShading = true
      }
    })

    model.position.set(0, 0, 0)
    model.scale.set(1, 1, 1)
  })

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 2)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 2)
  directionalLight.position.set(5, 10, 7)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.set(2048, 2048)
  directionalLight.shadow.camera.near = 0.5
  directionalLight.shadow.camera.far = 50
  directionalLight.shadow.camera.left = -10
  directionalLight.shadow.camera.right = 10
  directionalLight.shadow.camera.top = 10
  directionalLight.shadow.camera.bottom = -10
  scene.add(directionalLight)

  const fillLight = new THREE.PointLight(0xffffff, 0.4, 50)
  fillLight.position.set(-5, 5, -5)
  scene.add(fillLight)

  // Start animation
  animate()

  // Events
  window.addEventListener('resize', onWindowResize)
  window.addEventListener('scroll', onScrollTriggerRotate)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize)
  window.removeEventListener('scroll', onScrollTriggerRotate)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})

// Window resize
function onWindowResize() {
  if (!threeContainer.value) return

  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

// Scroll trigger: wanneer model moet draaien
function onScrollTriggerRotate() {
  const triggerScroll = 1000 // px

  if (window.scrollY > triggerScroll) {
    shouldRotate.value = true
  } else {
    shouldRotate.value = false
  }

  // Dynamisch schalen op basis van scroll (optioneel)
  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))
  targetScale = THREE.MathUtils.lerp(1, 2.5, scrollProgress)
}

// Animate
function animate() {
  animationId = requestAnimationFrame(animate)

  if (model) {
    // Rotatie als trigger actief is
    if (shouldRotate.value) {
      model.rotation.y += 0.03
    }

    // Floating effect
    floatTime += 0.02
    model.position.y = Math.sin(floatTime) * 0.1

    // Schaal animatie
    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)
  }

  renderer.render(scene, camera)
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





