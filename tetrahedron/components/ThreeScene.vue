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
  // scene.background = new THREE.Color('#ffffff')

  // Camera instellen (default FOV, wordt ook bij resize aangepast)
  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = window.innerWidth < 768 ? 6.5 : 5


  // Renderer instellen
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 2


  // 🔑 Belangrijk voor scherpe weergave op mobiel (retina schermen)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

// HDRI environment LIGHTING
const rgbeLoader = new RGBELoader()
rgbeLoader.load('/lighting.hdr', texture => {
  texture.mapping = THREE.EquirectangularReflectionMapping

  scene.environment = texture // reflecties
  scene.background = new THREE.Color('#FFDFF0') // neutrale achtergrond
})

  // Licht toevoegen aan de scene
setupLights(scene)

  // Licht toevoegen aan de scene
function setupLights(scene) {
  // zacht basislicht (vult schaduwen een beetje op)
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4)
  scene.add(ambientLight)

  // hoofdlicht, alsof het de zon is
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 5) // positie (x,y,z)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  // accentlicht van voren
  const pointLight = new THREE.PointLight(0xffffff, 0.6)
  pointLight.position.set(-5, 5, -5)
  scene.add(pointLight)

  // optioneel: klein randlicht voor glans
  const rimLight = new THREE.SpotLight(0xffffff, 0.8, 50, Math.PI / 6, 0.25, 1)
  rimLight.position.set(0, 5, 10)
  scene.add(rimLight)
}


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

    // pak de originele kleur van het glb-materiaal
    const origColor = child.material.color.clone()

    child.material = new THREE.MeshPhysicalMaterial({
      color: origColor, // behoudt de glb-kleur
      // metalness: 0.8,
      roughness: 0.2,
      clearcoat: 1,
      clearcoatRoughness: 0.05,
      flatShading: false,
      envMapIntensity: 1.5
    })
  }
})



    // startpositie model
    model.position.set(0, 0, 0)

    // 🔑 Schaal afhankelijk van schermbreedte
    const scaleFactor = window.innerWidth < 768 ? 1.5 : 2
    model.scale.set(scaleFactor, scaleFactor, scaleFactor)
  })

  // animatie starten
  animate()
  window.addEventListener('resize', onWindowResize)

  // initial resize uitvoeren zodat alles klopt bij eerste load
  onWindowResize()
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

  // 🔑 Camera FOV aanpassen voor mobiel (iets wijder zodat object in beeld blijft)
  if (width < 768) {
    camera.fov = 85
  } else {
    camera.fov = 75
  }
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

  // 🔑 Op mobiel scrollt men vaak minder → iets gevoeliger maken
  const scrollProgress = Math.min(
    1,
    Math.max(0, scrollTop / (docHeight * (window.innerWidth < 768 ? 0.6 : 1)))
  )

  // grenzen & oscillatie
  const visibleLimit = window.innerWidth < 768 ? 1.8 : 5
  const outOfViewOffset = window.innerWidth < 768 ? 0.5 : 1.5


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

  /* 🔑 Gebruik dynamic viewport height → beter voor mobiel/iOS Safari */
  height: 100dvh;

  pointer-events: none; /* klikken gaat erdoorheen */
  z-index: 10;
}
</style>









