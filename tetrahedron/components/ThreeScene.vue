<template>
  <div>
    <section class="three-container" ref="threeContainer"></section>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

const threeContainer = ref(null)

// basis Three.js objecten
let scene, camera, renderer, animationId
let model = null

// animatie-doelen 
let targetX = 0
let targetY = 0
let targetRotationY = 0
let targetRotationZ = 0
let targetScale = 1

// voor "float"-effect
let floatTime = 0

// scroll-pauze variabelen
let pauseUntil = 0
const pauseDuration = 1200 // 1.2s stilstaan
let pauseScrollY = 0       // onthoud scrollTop bij start pauze

onMounted(() => {
  window.addEventListener('scroll', handleScroll)

  // Scene maken
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#F8F8FF')

  // Camera instellen
  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5

  // Renderer instellen
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

  // HDR / belichting config laden
  fetch('/lighting2.json')
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

  // 3D model laden
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', gltf => {
    model = gltf.scene
    scene.add(model)

// door alle meshes lopen → schaduwen & materiaal instellen
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

  // animatie starten
  animate()
  window.addEventListener('resize', onWindowResize)
})

onBeforeUnmount(() => {
  // listeners opruimen
  window.removeEventListener('scroll', handleScroll)
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})

// herbereken canvas bij resize
function onWindowResize() {
  if (!threeContainer.value) return
  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight
  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

// animatie-loop
function animate() {
  animationId = requestAnimationFrame(animate)

  if (model) {
    // model beweegt langzaam richting target waardes
    model.position.x += (targetX - model.position.x) * 0.1
    model.position.y = THREE.MathUtils.lerp(model.position.y, targetY, 0.1)

    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07

    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    // zweef-effect (sinus golf)
    floatTime += 0.01
    model.position.y = Math.sin(floatTime) * 0.1
  }

  renderer.render(scene, camera)
}

// scroll event → bepaalt target-waardes
function handleScroll() {
  // check of pauze actief is
  if (Date.now() < pauseUntil) {
    // scroll terugzetten naar punt van pauze → "freeze" pagina
    window.scrollTo(0, pauseScrollY)
    return
  }

  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))

  // grenzen & oscillatie
  const visibleLimit = 5
  const outOfViewOffset = 1.5
  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  const oscillationCount = 5
  const oscillation = Math.sin(scrollProgress * Math.PI * oscillationCount)

  // target posities/rotaties schalen met scroll
  targetX = THREE.MathUtils.lerp(leftEndX, rightEndX, (oscillation + 1) / 2)
  const maxY = 2
  targetY = (targetX / rightEndX) * maxY

  targetRotationY = oscillation * (Math.PI / 2)
  targetRotationZ = scrollProgress * Math.PI * 2
  targetScale = 1.8

  // check of we bij links/rechts zijn → pauze starten
  if (
    Math.abs(targetX - rightEndX) < 0.05 ||
    Math.abs(targetX - leftEndX) < 0.05
  ) {
    pauseUntil = Date.now() + pauseDuration
    pauseScrollY = scrollTop // sla huidige scrollpositie op
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
  inset: 0; /* zelfde als top/left/right/bottom = 0 */
  width: 100vw;
  height: 100vh;
  pointer-events: none; /* klikken gaat erdoorheen */
  z-index: 10;
}
</style>






