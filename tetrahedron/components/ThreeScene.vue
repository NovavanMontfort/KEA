<template>
  <div ref="threeContainer" class="three-container"></div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const threeContainer = ref(null)

let scene, camera, renderer, animationId
let model = null

onMounted(() => {
  // Create scene and camera
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x202020)

  camera = new THREE.PerspectiveCamera(
    75,
    threeContainer.value.clientWidth / threeContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.z = 5

  // Create renderer
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(
    threeContainer.value.clientWidth,
    threeContainer.value.clientHeight
  )
  threeContainer.value.appendChild(renderer.domElement)

  // Load GLB model
  const loader = new GLTFLoader()
  loader.load(
    '/tetrahedron.glb',
    (gltf) => {
      model = gltf.scene
      scene.add(model)
      model.position.set(0, 0, 0)
      model.scale.set(1, 1, 1)
    },
    // (xhr) => {
    //   console.log((xhr.loaded / xhr.total) * 100 + '% loaded')
    // },
    // (error) => {
    //   console.error('An error happened loading the model', error)
    // }
  )

  // Animation loop
  const animate = () => {
    animationId = requestAnimationFrame(animate)
    if (model) {
      model.rotation.x += 0.01
      model.rotation.y += 0.01
    }
    renderer.render(scene, camera)
  }
  animate()

  // Resize listener
  window.addEventListener('resize', onWindowResize)
})

onBeforeUnmount(() => {
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
</script>

<style scoped>
html, body, #__nuxt, #app {
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

.three-container {
  width: 100vw;
  height: 100vh;
  /* border: 2px solid red; */
}
</style>


