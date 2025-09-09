<template>
  <div>
    <section style="height: 100vh; background: #111; color: white;">
      <h1>Welkom bovenaan</h1>
    </section>

    <section class="three-container" ref="threeContainer"></section> <!-- Hier staat je model -->

    <section style="height: 2000px; background: #eee;">
      <p>Meer content onder het 3D-model...</p>
    </section>
  </div>
</template>


<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const threeContainer = ref(null)

let scene, camera, renderer, animationId
let model = null
let isScrolling = false
let scrollTimeout = null
let modelX = 0
let direction = 1 // 1 = naar rechts, -1 = naar links
let lastScrollY = 0
const xLimit = 4.5 // Max. positie naar links/rechts (aangepast op schermbreedte)



onMounted(() => {

  // 1. Maak een nieuwe 3D scene aan
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020) // Donkere achtergrondkleur

  // 2. Stel de camera in (hoe en wat er zichtbaar is)
  camera = new THREE.PerspectiveCamera(
    75, // Field of view
    threeContainer.value.clientWidth / threeContainer.value.clientHeight, // Aspect ratio
    0.1, // Nabijste zichtbare afstand
    1000 // Verste zichtbare afstand
  )
  camera.position.z = 5 // Zet de camera iets naar achter

  // 3. Renderer: dit maakt het beeld zichtbaar op het scherm
  renderer = new THREE.WebGLRenderer({ antialias: true }) // Antialiasing maakt lijnen gladder
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement) // Voeg de canvas toe aan de pagina

  // 4. Laad het 3D model (GLB-bestand uit de public map)
  const loader = new GLTFLoader()
  loader.load(
    '/tetrahedron.glb', // Bestand moet in /public map staan
    (gltf) => {
      model = gltf.scene // Haal de 3D scene uit het bestand
      scene.add(model) // Voeg model toe aan de 3D scene
      model.position.set(0, 0, 0) // Zet model in het midden
      model.scale.set(1, 1, 1) // Schaal het model (optioneel)
    },
  )

  // 5. Voeg belichting toe aan de scene

  // Algemeen omgevingslicht – zorgt ervoor dat alles een beetje zichtbaar is
  const ambientLight = new THREE.AmbientLight(0xffffff, 1)
  scene.add(ambientLight)

  // Gericht licht – alsof er een zon of lamp schijnt
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 7) // Richting en positie van het licht
  scene.add(directionalLight)

  // 6. Start de animatie-loop (herhaalt zichzelf elke frame)
  const animate = () => {
  animationId = requestAnimationFrame(animate)

  if (model && isScrolling) {
    model.rotation.x += 0.01
    model.rotation.y += 0.01
  }

  renderer.render(scene, camera)
}

  animate()

  // 7. Pas renderer en camera aan als het schermformaat verandert
  window.addEventListener('resize', onWindowResize)

  window.addEventListener('scroll', handleScroll) 
  lastScrollY = window.scrollY

})

onBeforeUnmount(() => {
  // Opruimen: verwijder events en stop animatie
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationId)
  renderer.dispose()

  window.removeEventListener('scroll', handleScroll)
})

// Wordt aangeroepen als het scherm wordt vergroot/verkleind
function onWindowResize() {
  if (!threeContainer.value) return

  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix() // Update de camera-instellingen
  renderer.setSize(width, height) // Pas canvasgrootte aan
}

function handleScroll() {
  const scrollDelta = window.scrollY - lastScrollY
  lastScrollY = window.scrollY

  // Scrollactiviteit bijhouden
  isScrolling = true
  if (scrollTimeout !== null) clearTimeout(scrollTimeout)
  scrollTimeout = setTimeout(() => (isScrolling = false), 100)

  if (!model) return

  // Beweeg model horizontaal afhankelijk van richting
  modelX += direction * 0.05 * Math.sign(scrollDelta)

  // Bots met zijkanten? Dan keer richting om
  if (modelX > xLimit) {
    modelX = xLimit
    direction = -1
  } else if (modelX < -xLimit) {
    modelX = -xLimit
    direction = 1
  }

  // Zet modelpositie
  model.position.x = modelX
  model.position.y = 0 // optioneel: vaste Y
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
  pointer-events: none; /* zodat scroll en clicks door kunnen */
  z-index: 10;
}

</style>



