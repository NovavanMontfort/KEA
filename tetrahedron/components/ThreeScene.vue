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

// GSAP + ScrollTrigger
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
gsap.registerPlugin(ScrollTrigger)

// --- debug toggle: zet op true om markers te zien in de pagina (handig bij troubleshooting)
const DEBUG = false

const threeContainer = ref(null)

// Three.js essentials
let scene, camera, renderer, animationId
let model = null
let scrollTimeline = null

onMounted(() => {
  // Scene
  scene = new THREE.Scene()

  // Camera
  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = window.innerWidth < 768 ? 6.5 : 5

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 2
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

  // HDRI (asynchroon)
  const rgbeLoader = new RGBELoader()
  rgbeLoader.load('/lighting.hdr', texture => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    scene.background = new THREE.Color('#FFDFF0')
  }, undefined, err => {
    // niet fatal — alleen loggen
    if (DEBUG) console.warn('HDR load error:', err)
  })

  setupLights(scene)

  // GLTF laden
  const loader = new GLTFLoader()
  loader.load('/tetrahedron.glb', gltf => {
    model = gltf.scene
    scene.add(model)

    model.traverse(child => {
      if (child.isMesh) {
        child.castShadow = true
        child.receiveShadow = true
        const origColor = child.material.color ? child.material.color.clone() : new THREE.Color(0xffffff)
        child.material = new THREE.MeshPhysicalMaterial({
          color: origColor,
          roughness: 0.2,
          clearcoat: 1,
          clearcoatRoughness: 0.05,
          envMapIntensity: 1.5
        })
      }
    })

    // init schaal & positie
    const scaleFactor = window.innerWidth < 768 ? 1.5 : 2
    model.scale.set(scaleFactor, scaleFactor, scaleFactor)
    model.position.set(0, 0, 0)

    // start GSAP animatie
    initScrollAnimation(model)
    // force a refresh after animatie creation
    ScrollTrigger.refresh()
  }, undefined, err => {
    console.error('GLTF load error:', err)
  })

  // start render loop
  animate()
  window.addEventListener('resize', onWindowResize)
  onWindowResize()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationId)

  // kill GSAP/ScrollTrigger
  if (scrollTimeline) {
    scrollTimeline.scrollTrigger && scrollTimeline.scrollTrigger.kill && scrollTimeline.scrollTrigger.kill()
    scrollTimeline.kill && scrollTimeline.kill()
  }
  // kill any remaining ScrollTriggers
  ScrollTrigger.getAll().forEach(st => st.kill())

  // dispose renderer
  renderer.dispose()
})

function setupLights(scene) {
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 5)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  const pointLight = new THREE.PointLight(0xffffff, 0.6)
  pointLight.position.set(-5, 5, -5)
  scene.add(pointLight)

  const rimLight = new THREE.SpotLight(0xffffff, 0.8, 50, Math.PI / 6, 0.25, 1)
  rimLight.position.set(0, 5, 10)
  scene.add(rimLight)
}

function onWindowResize() {
  if (!threeContainer.value) return
  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.fov = width < 768 ? 85 : 75
  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)

  // refresh ScrollTrigger ranges
  ScrollTrigger.refresh()
}

function animate() {
  animationId = requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

/**
 * Robustere ScrollTrigger setup:
 * - trigger = document.body (zodat de volledige pagina-scroll gebruikt wordt)
 * - end = dynamisch: als pagina geen scroll heeft -> we creëren een minimale scrollafstand
 * - immediateRender: false zodat model niet “teleporteert” bij init
 */
function initScrollAnimation(model) {
  // safety
  if (!model) {
    console.warn('initScrollAnimation called but model is null')
    return
  }

  const visibleLimit = window.innerWidth < 768 ? 1.8 : 5
  const outOfViewOffset = window.innerWidth < 768 ? 0.5 : 1.5
  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  // bereken scrollLength: als pagina niet scrollbaar is, maak een fallback
  const docHeight = document.documentElement.scrollHeight
  const winH = window.innerHeight
  let scrollLength = Math.max(1600, docHeight - winH) // minstens 1600px scroll-range
  // als document daadwerkelijk groter is dan viewport, gebruik die waarde
  if (docHeight - winH > 0) scrollLength = docHeight - winH

  if (DEBUG) {
    console.log('docHeight, winH, scrollLength', docHeight, winH, scrollLength)
  }

  // kill vorige timeline als die er was
  if (scrollTimeline) {
    scrollTimeline.kill && scrollTimeline.kill()
  }

  scrollTimeline = gsap.timeline({
    defaults: { immediateRender: false },
    scrollTrigger: {
      trigger: document.body, // gebruik de pagina-scroll
      start: 'top top',
      end: () => '+=' + scrollLength, // dynamische end (px)
      scrub: true,
      // zet markers: true tijdens debugging om start/end te zien
      markers: DEBUG
    }
  })

  // X beweging (van links naar rechts)
  scrollTimeline.fromTo(model.position,
    { x: leftEndX },
    { x: rightEndX, ease: 'none', immediateRender: false },
    0
  )

  // Y (licht omhoog)
  scrollTimeline.to(model.position, { y: 2, ease: 'sine.inOut', immediateRender: false }, 0)

  // Rotaties
  scrollTimeline.to(model.rotation, {
    y: Math.PI / 2,
    z: Math.PI * 2,
    ease: 'none',
    immediateRender: false
  }, 0)

  // Schaal
  scrollTimeline.to(model.scale, {
    x: 1.8,
    y: 1.8,
    z: 1.8,
    ease: 'power1.inOut',
    immediateRender: false
  }, 0)

  // Optioneel: kleine 'zweef' animatie die los staat van scroll
  // (comment uit als je dat niet wil)
  gsap.to(model.position, {
    y: '+=0.12',
    duration: 2.8,
    yoyo: true,
    repeat: -1,
    ease: 'sine.inOut',
    paused: false
  })
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

  /* fallback first */
  height: 100vh;
  /* modern browsers that support dvh will override */
  height: 100dvh;

  pointer-events: none;
  z-index: 10;
}
</style>












