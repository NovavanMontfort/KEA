<template>
  <div>
    <!-- Container voor de 3D scene -->
    <section class="three-container" ref="threeContainer"></section>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

/**
 * DOM referentie waar het Three.js canvas in wordt geplaatst
 */
const threeContainer = ref(null)

/**
 * Basis Three.js objecten
 */
let scene, camera, renderer, animationId
let model = null

/**
 * Animatie-doelen (hier beweegt/scaled/roteert het model naartoe)
 */
let targetX = 0
let targetY = 0
let targetRotationY = 0
let targetRotationZ = 0
let targetScale = 1

// Variabele om een “zweef”-effect te berekenen
let floatTime = 0

// Scroll-pauze variabelen → zorgen ervoor dat het model soms even stil blijft staan
let pauseUntil = 0
const pauseDuration = 1200 // pauze van 1.2 seconden
let pauseScrollY = 0       // onthoudt scrollpositie bij start van de pauze

onMounted(() => {
  // Luister naar scroll events
  window.addEventListener('scroll', handleScroll)

  // Scene maken (dit is de “wereld” waar alles in staat)
  scene = new THREE.Scene()
  scene.background = new THREE.Color('#ffffff') // standaard achtergrondkleur

  // Camera instellen (bepaalt vanuit welk punt je kijkt)
  camera = new THREE.PerspectiveCamera(
  75, // Field of View (hoe “wijd” de camera kijkt)
  threeContainer.value.clientWidth / threeContainer.value.clientHeight, // aspect ratio → breedte/hoogte van het canvas
  0.1, // dichtbij-plane → alles dichterbij dan 0.1 wordt niet getekend
  1000 // verweg-plane → alles verder weg dan 1000 wordt niet getekend
  )
  // Zet de camera iets verder weg op mobiel, zodat het object goed in beeld blijft
  camera.position.z = window.innerWidth < 768 ? 6.5 : 5


  // Renderer instellen (tekenen naar het canvas in de browser)
  renderer = new THREE.WebGLRenderer({ antialias: true }) // maakt randen gladder
  renderer.shadowMap.enabled = true // schaduwen aanzetten
  renderer.shadowMap.type = THREE.PCFSoftShadowMap // zachtere schaduwen

  renderer.toneMapping = THREE.ACESFilmicToneMapping // mooiere belichting (filmisch)
  renderer.toneMappingExposure = 2 // belichting iets sterker maken

  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)) // zorgt voor scherp beeld op retina-schermen
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  // Voeg het canvas toe aan de HTML (anders zie je niks)
  threeContainer.value.appendChild(renderer.domElement)


  // HDRI environment → zorgt voor realistische belichting/reflecties
  const rgbeLoader = new RGBELoader()
  rgbeLoader.load('/lighting.hdr', texture => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color('#FFDFF0') // neutrale achtergrond
  })

  // Basis belichting toevoegen
  setupLights(scene)

  // 3D model laden (GLB bestand)
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', gltf => {
    model = gltf.scene
    scene.add(model)

    // Loop door alle meshes in het model
    model.traverse(child => {
      if (child.isMesh) {
        child.castShadow = true
        child.receiveShadow = true

        // originele kleur uit bestand ophalen
        const origColor = child.material.color.clone()

        // nieuw fysiek materiaal toewijzen (glans/reflexie)
        child.material = new THREE.MeshPhysicalMaterial({
          color: origColor,
          roughness: 0.2,
          clearcoat: 1,
          clearcoatRoughness: 0.05,
          flatShading: false,
          envMapIntensity: 1.5
        })
      }
    })

  // Startpositie & schaal instellen
  model.position.set(0, 0, 0) // zet het model in het midden
  // Op mobiel iets kleiner maken, zodat het in beeld past
  const scaleFactor = window.innerWidth < 768 ? 1.5 : 2
  model.scale.set(scaleFactor, scaleFactor, scaleFactor)

  })

  // Start animatie-loop
  animate()

  // Luister naar window-resizes
  window.addEventListener('resize', onWindowResize)
  onWindowResize() // eerste keer direct uitvoeren
})

onBeforeUnmount(() => {
  // Opruimen wanneer component verdwijnt
  window.removeEventListener('scroll', handleScroll)
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationId)
  renderer.dispose()
})

/**
 * Functie om licht toe te voegen aan de scene
 */
function setupLights(scene) {
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4) // zacht basislicht
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1) // zonlicht
  directionalLight.position.set(5, 10, 5)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  const pointLight = new THREE.PointLight(0xffffff, 0.6) // extra licht van voren
  pointLight.position.set(-5, 5, -5)
  scene.add(pointLight)

  const rimLight = new THREE.SpotLight(0xffffff, 0.8, 50, Math.PI / 6, 0.25, 1) // randlicht voor glans
  rimLight.position.set(0, 5, 10)
  scene.add(rimLight)
}

/**
 * Functie om canvas & camera aan te passen bij window resize
 */
function onWindowResize() {
  if (!threeContainer.value) return
  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  // andere camera-instelling voor mobiel
  camera.fov = width < 768 ? 85 : 75 // Pas Field of View aan: mobiel krijgt iets wijdere lens
  camera.aspect = width / height // verhouding aanpassen aan scherm
  camera.updateProjectionMatrix() // herbereken projectie

  renderer.setSize(width, height) // canvas opnieuw schalen
}

/**
 * Animatie-loop → draait elke frame
 */
function animate() {
  animationId = requestAnimationFrame(animate)

  if (model) {
    // Positie en rotatie bewegen langzaam naar target waarden
    // X-positie geleidelijk naar targetX
    model.position.x += (targetX - model.position.x) * 0.1
    // Y-positie vloeiend naar targetY
    model.position.y = THREE.MathUtils.lerp(model.position.y, targetY, 0.1)

    // Rotaties langzaam bijwerken
    model.rotation.y += (targetRotationY - model.rotation.y) * 0.07
    model.rotation.z += (targetRotationZ - model.rotation.z) * 0.07

    // Schaal langzaam aanpassen naar targetScale
    const currentScale = model.scale.x
    const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
    model.scale.set(newScale, newScale, newScale)

    // zweef-effect
    floatTime += 0.01
    model.position.y = Math.sin(floatTime) * 0.1
  }

  // Scene tekenen
  renderer.render(scene, camera)
}

/**
 * Scroll event → bepaalt hoe het model beweegt bij scrollen
 */
function handleScroll() {
  // Als er een pauze actief is → bevries de scrollpositie
  if (Date.now() < pauseUntil) {
    window.scrollTo(0, pauseScrollY)
    return
  }

  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight

  // Op mobiel iets gevoeliger scrollen
  const scrollProgress = Math.min(
  1, // max 1
  Math.max(0, scrollTop / (docHeight * (window.innerWidth < 768 ? 0.6 : 1)))
  )
  // → scrollProgress loopt van 0 (bovenaan) tot 1 (onderaan)
  // Op mobiel iets gevoeliger gemaakt (0.6)


  // Grenzen waarbinnen het model mag bewegen
  // Grenzen instellen (verschil desktop/mobiel)
  const visibleLimit = window.innerWidth < 768 ? 1.8 : 5
  const outOfViewOffset = window.innerWidth < 768 ? 0.5 : 1.5
  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

// Hoe vaak het model heen-en-weer slingert tijdens de volledige scroll
const oscillationCount = 5 

// Maak een golf (sinus) die soepel heen-en-weer beweegt tussen -1 en 1
const oscillation = Math.sin(scrollProgress * Math.PI * oscillationCount)

// Horizontale positie (X-as):
// - lerp() kiest een punt tussen leftEndX en rightEndX
// - (oscillation + 1) / 2 zet de sinus (-1..1) om naar een bruikbare waarde (0..1)
targetX = THREE.MathUtils.lerp(leftEndX, rightEndX, (oscillation + 1) / 2)

// Verticale positie (Y-as):
// - maxY bepaalt hoe hoog het model maximaal beweegt
// - positie is gekoppeld aan targetX, zodat hij iets op en neer zweeft tijdens het heen-en-weer schuiven
const maxY = 2
targetY = (targetX / rightEndX) * maxY

// Rotatie om de Y-as (links/rechts draaien):
// - oscillation bepaalt hoe ver het draait
// - Math.PI/2 = maximaal 90° draai
targetRotationY = oscillation * (Math.PI / 2)

// Rotatie om de Z-as (rollen/spinnen):
// - recht evenredig met scrollProgress
// - Math.PI * 2 = precies één volledige spin tijdens de hele scroll
targetRotationZ = scrollProgress * Math.PI * 2

// Schaal grootte:
// - hier vastgezet op 1.8x
// - zou je dit dynamisch maken met scrollProgress, dan groeit het model tijdens scroll
targetScale = 1.8

// Check of het model bijna aan de linker- of rechterrand is
// - Math.abs(...) < 0.05 = kleine marge om te bepalen of het "bij de rand" staat
// Als dat zo is → zet een pauze in de animatie
if (
  Math.abs(targetX - rightEndX) < 0.05 || // bij de rechterrand
  Math.abs(targetX - leftEndX) < 0.05    // bij de linkerrand
) {
  // Pauzeer animatie tot (nu + pauseDuration)
  pauseUntil = Date.now() + pauseDuration

  // Onthoud huidige scrollpositie, zodat de pagina niet verder kan scrollen tijdens de pauze
  pauseScrollY = scrollTop
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
  inset: 0;
  width: 100vw;

  height: 100dvh;

  pointer-events: none; 
  z-index: 10;
}
</style>










