# Vue + Three.js Scroll Animation Demo

This project is a Vue 3 component that renders a 3D model using [Three.js](https://threejs.org/), with smooth scroll-driven animations. It loads a GLB model and animates it based on the user's scroll position, including:

- Horizontal movement
- Y-axis rotation (left/right)
- Z-axis "roll" (like a somersault) (extra)
- Smooth scaling
- A subtle floating animation


## Features

- Loads a `.glb` 3D model (`/tetrahedron.glb`)
- Animations linked to scroll progress
- Floating effect for a more dynamic look (extra)
- Responsive resizing


## Install dependencies

This assumes you're working inside a Vue 3 project using Vite or Nuxt 3.

```bash
npm install three
