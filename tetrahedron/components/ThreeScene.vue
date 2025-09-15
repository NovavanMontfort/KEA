<template>
  <div>
    <section style="height: 100vh;"></section>

    <section class="three-container" ref="threeContainer"></section>

    <section style="height: 5000px;"></section>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const threeContainer = ref(null)

let scene, camera, renderer, animationId
let model = null

let targetX = 0
let targetY = 0
let targetRotationY = 0
let targetRotationZ = 0
let targetScale = 1

let floatTime = 0

let isScrolling = false
let scrollTimeout = null

const visibleLimit = 3 // Maximale X-positie binnen het zichtbare gebied
const outOfViewOffset = 1.5 // Extra offset om buiten beeld te gaan

onMounted(() => {

  window.addEventListener('scroll', handleScroll)

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
        child.material.needsUpdate = true
      }
    })

    model.position.set(0, 0, 0)
    model.scale.set(2, 2, 2)
  })

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.2)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 5)
  directionalLight.position.set(3, 10, 5) 

  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.set(2048, 2048)
  directionalLight.shadow.camera.near = 0.5
  directionalLight.shadow.camera.far = 50
  directionalLight.shadow.camera.left = -10
  directionalLight.shadow.camera.right = 10
  directionalLight.shadow.camera.top = 10
  directionalLight.shadow.camera.bottom = -10
  scene.add(directionalLight)

  animate()

  window.addEventListener('resize', onWindowResize)

})

onBeforeUnmount(() => {
  window.removeEventListener('scroll', handleScroll)

  window.removeEventListener('resize', onWindowResize)

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
    model.position.x += (targetX - model.position.x) * 0.1
    model.position.y = THREE.MathUtils.lerp(model.position.y, targetY, 0.1) + Math.sin(floatTime) * 0.1;

    
    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07

    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    // EXTRA - "koprol" animatie
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07

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

  // Hoe ver het model mag bewegen horizontaal
  const visibleLimit = 5
  const outOfViewOffset = 1.5
  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  // Hoe vaak het model heen en weer moet bewegen
  const oscillationCount = 5
  const oscillation = Math.sin(scrollProgress * Math.PI * oscillationCount)

  // X-positie tussen links en rechts (heen en weer)
  targetX = THREE.MathUtils.lerp(leftEndX, rightEndX, (oscillation + 1) / 2)

  // Y-positie: lineair schuin bewegen (diagonaal)
  // Als targetX helemaal rechts is, dan is targetY maxY (boven)
  // Als targetX helemaal links is, dan is targetY -maxY (onder)
  const maxY = 8
  targetY = (targetX / rightEndX) * maxY

  // Rotaties zoals voorheen
  targetRotationY = oscillation * (Math.PI / 2)
  targetRotationZ = scrollProgress * Math.PI * 2

  // Schaal vastgezet op maxScale (kan je aanpassen)
  const minScale = 1
  const maxScale = 1.8
  targetScale = maxScale
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




