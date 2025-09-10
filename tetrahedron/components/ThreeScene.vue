<template>
  <div>
    <section style="height: 100vh;"></section>

    <section class="three-container" ref="threeContainer"></section>

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

let scrollTimeout = null
let isScrolling = false

let targetX = 0
let targetRotationY = 0
let targetRotationZ = 0     // EXTRA - "koprol" animatie
let targetScale = 1

let floatTime = 0;    // EXTRA - floating animatie

const visibleLimit = 4.5
const outOfViewOffset = 1.5

onMounted(() => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020)

  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', (gltf) => {
    model = gltf.scene
    scene.add(model)
    model.position.set(0, 0, 0)
    model.scale.set(2, 2, 2)
  })

  const ambientLight = new THREE.AmbientLight(0xffffff, 1)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 7)
  scene.add(directionalLight)

  animate()

  window.addEventListener('resize', onWindowResize)
  window.addEventListener('scroll', handleScroll)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize)
  window.removeEventListener('scroll', handleScroll)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})

function onWindowResize() {
  if (!threeContainer.value) return

  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

function animate() {
  animationId = requestAnimationFrame(animate)

  if (model) {
    // Positie op de X-as wordt soepel geïnterpoleerd
    model.position.x += (targetX - model.position.x) * 0.1

    // Y-rotatie blijft zoals je eerder hebt ingesteld
    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07

    // EXTRA - "koprol" animatie
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07
 
    // Scale interpolatie
    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    // EXTRA - floating animatie
    floatTime += 0.01; // Pas de snelheid van de beweging aan door de waarde te verhogen of te verlagen
    model.position.y = Math.sin(floatTime) * 0.1; // 0.1 is de amplitude van de beweging, pas dit aan naar wens
  }

  renderer.render(scene, camera)
}


function handleScroll() {
  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))

  isScrolling = true
  if (scrollTimeout !== null) clearTimeout(scrollTimeout)
  scrollTimeout = setTimeout(() => (isScrolling = false), 100)

  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  if (scrollProgress <= 0.5) {
    const t = scrollProgress * 2
    targetX = THREE.MathUtils.lerp(0, rightEndX, t)
  } else {
    const t = (scrollProgress - 0.5) * 2
    targetX = THREE.MathUtils.lerp(rightEndX, leftEndX, t)
  }

  // Y-rotatie: heen en terug symmetrisch
  targetRotationY = Math.sin(scrollProgress * Math.PI) * (Math.PI / 2)

    // Z-rotatie: laat ‘m een ronde maken op scroll
  targetRotationZ = scrollProgress * Math.PI * 2

  // Schaal
  targetScale = THREE.MathUtils.lerp(1, 3, scrollProgress)
}
</script>

<style scoped>
html, body, #__nuxt, #app {
  height: 100%;
  margin: 0;
  padding: 0;
}

.three-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 10;
}
</style>




