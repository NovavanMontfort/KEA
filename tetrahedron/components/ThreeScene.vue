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
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

const threeContainer = ref(null)

let scene, camera, renderer, animationId
let model = null

let targetX = 0
let targetY = 0
let targetRotationY = 0
let targetRotationZ = 0
let targetScale = 1

let floatTime = 0

// pauze-variabelen
let pauseUntil = 0
const pauseDuration = 1200 // ms = 1.2s stil blijven staan

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

  // HDR / lighting
  fetch('/lighting.json')
    .then(res => res.json())
    .then(config => {
      const hdrUrl = config.hdr.texture_url.replace(/\[size\]/g, config.hdr.size)

      renderer.toneMapping = THREE.CineonToneMapping
      renderer.toneMappingExposure = config.global.toneMappingExposure || 1

      const rgbeLoader = new RGBELoader()
      rgbeLoader.load(hdrUrl, texture => {
        texture.mapping = THREE.EquirectangularReflectionMapping

        if (config.hdr.use_as_background) scene.background = texture
        if (config.hdr.use_as_environment) scene.environment = texture

        console.log('HDR geladen & toegepast')
      })
    })
    .catch(err => console.error('Fout bij laden van lighting.json:', err))

  // Model
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', gltf => {
    model = gltf.scene
    scene.add(model)

    model.traverse(child => {
      if (child.isMesh) {
        child.castShadow = true
        child.receiveShadow = true
        child.material.flatShading = true
        child.material.envMapIntensity = 2
        child.material.needsUpdate = true
      }
    })

    model.position.set(0, 0, 0)
    model.scale.set(2, 2, 2)
  })

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
    model.position.y = THREE.MathUtils.lerp(model.position.y, targetY, 0.1)

    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07

    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    floatTime += 0.01
    model.position.y = Math.sin(floatTime) * 0.1
  }

  renderer.render(scene, camera)
}

function handleScroll() {
  // als we nog in een pauze zitten → skip updates
  if (Date.now() < pauseUntil) return

  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))

  const visibleLimit = 5
  const outOfViewOffset = 1.5
  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  const oscillationCount = 5
  const oscillation = Math.sin(scrollProgress * Math.PI * oscillationCount)

  targetX = THREE.MathUtils.lerp(leftEndX, rightEndX, (oscillation + 1) / 2)
  const maxY = 2
  targetY = (targetX / rightEndX) * maxY

  targetRotationY = oscillation * (Math.PI / 2)
  targetRotationZ = scrollProgress * Math.PI * 2
  targetScale = 1.8

  // 🚩 check of we bij links/rechts zijn → pauze starten
  if (Math.abs(targetX - rightEndX) < 0.05 || Math.abs(targetX - leftEndX) < 0.05) {
    pauseUntil = Date.now() + pauseDuration
  }
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
  inset: 0; /* korter dan top/left/width/height */
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 10;
}
</style>






