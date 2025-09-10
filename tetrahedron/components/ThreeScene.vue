<template>
  <div>
    <section style="height: 100vh;">
    </section>

    <section class="three-container" ref="threeContainer"></section> <!-- Hier staat je model -->

    <section style="height: 2000px;">
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

let lastScrollY = 0
let isScrolling = false
let scrollTimeout = null

// Target waarden waar het model naar toe beweegt
let targetX = 0
let targetRotationY = 0
let targetScale = 1

const visibleLimit = 4.5       // zichtbare grens
const outOfViewOffset = 1.5   // extra om half van het model buiten beeld te laten

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
    model.scale.set(2, 2, 2) //sizing
  })

  //test

  const ambientLight = new THREE.AmbientLight(0xffffff, 1)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(5, 10, 7)
  scene.add(directionalLight)

  const animate = () => {
    animationId = requestAnimationFrame(animate)

    if (model) {
      // Smooth interpolatie naar target waarden
      model.position.x += (targetX - model.position.x) * 0.1
      model.rotation.y += (targetRotationY - model.rotation.y) * 0.07 // snelheid van draai

      // Extra rotatie als er gescrolled wordt
      if (isScrolling) {
      model.rotation.x += 0.01 // snelheid van draai
    }

      //smooth scale
      const currentScale = model.scale.x
      const newScale = THREE.MathUtils.lerp(currentScale, targetScale, 0.1)
      model.scale.set(newScale, newScale, newScale)
    }

    renderer.render(scene, camera)
  }

  animate()

  window.addEventListener('resize', onWindowResize)
  window.addEventListener('scroll', handleScroll)
  lastScrollY = window.scrollY
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize)
  cancelAnimationFrame(animationId)
  renderer.dispose()

  window.removeEventListener('scroll', handleScroll)
})

function onWindowResize() {
  if (!threeContainer.value) return

  const width = threeContainer.value.clientWidth
  const height = threeContainer.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

function handleScroll() {
  const scrollTop = window.scrollY
  const docHeight = document.body.scrollHeight - window.innerHeight
  const scrollProgress = Math.min(1, Math.max(0, scrollTop / docHeight))

  isScrolling = true
  if (scrollTimeout !== null) clearTimeout(scrollTimeout)
  scrollTimeout = setTimeout(() => (isScrolling = false), 100)

  // Scroll progress van 0 -> 1
  // 0 -> 0.5: model beweegt van midden naar rechts (met punt naar binnen)
  // 0.5 -> 1: model beweegt van rechts naar links (met punt naar binnen)

  const rightEndX = visibleLimit + outOfViewOffset
  const leftEndX = -visibleLimit - outOfViewOffset

  if (scrollProgress <= 0.5) {
    const t = scrollProgress * 2
    targetX = THREE.MathUtils.lerp(0, rightEndX, t)
    targetRotationY = THREE.MathUtils.lerp(0, Math.PI / 2, t)
  } else {
    const t = (scrollProgress - 0.5) * 2
    targetX = THREE.MathUtils.lerp(rightEndX, leftEndX, t)
    targetRotationY = THREE.MathUtils.lerp(Math.PI / 2, -Math.PI / 2, t)
  }

  // Schaal tussen 1 en 1.5 op basis van scrollprogress
  targetScale = THREE.MathUtils.lerp(1, 5, scrollProgress) //sizing
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



