<script setup>
import { onMounted } from 'vue'
import * as THREE from 'three'

onMounted(() => {
    // Threejs setup
    const canvas = document.querySelector('canvas')
    const scene = new THREE.Scene()
    const renderer = new THREE.WebGLRenderer({
        canvas: canvas,
        alpha: true
    })
    
    // Textures and loading
    // Earth textures: 
    // - https://planetpixelemporium.com/earth8081.html
    // - http://shadedrelief.com/natural3/ne3_data/8192/clouds/fair_clouds_8k.jpg
    const loadingManager = new THREE.LoadingManager()
    const textureLoader = new THREE.TextureLoader(loadingManager)
    loadingManager.onLoad = () => { 
        // Handle vuejs stuff
        console.log("Textures loaded!")
    }

    // Earth
    const earthMaterial = new THREE.MeshToonMaterial()
    earthMaterial.map = textureLoader.load('src/assets/earth3d/textures/earthmap.jpg')
    earthMaterial.bumpMap = textureLoader.load('src/assets/earth3d/textures/earthbump.jpg')
    earthMaterial.bumpScale = .005
    earthMaterial.metalnessMap = textureLoader.load('src/assets/earth3d/textures/earthmetalness.jpg')
    earthMaterial.roughnessMap = textureLoader.load('src/assets/earth3d/textures/earthroughness.jpg')
    earthMaterial.normalMap = textureLoader.load('src/assets/earth3d/textures/earthnormal.jpg')
    earthMaterial.normalScale = new THREE.Vector2(.1, .1)
    // earthMaterial.emissive = 0xaaddff
    // earthMaterial.emissiveIntensity = 0
    const earthMesh = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), earthMaterial)

    // Clouds
    const cloudsTexture = textureLoader.load('src/assets/earth3d/textures/clouds.jpg')
    const cloudsMaterial = new THREE.MeshToonMaterial({color: 0xacdfff, transparent: true, opacity: .7})
    cloudsMaterial.alphaMap = cloudsTexture
    cloudsMaterial.metalness = 0
    cloudsMaterial.roughness = 1
    const cloudsMesh = new THREE.Mesh(new THREE.SphereGeometry(1.02, 32, 32), cloudsMaterial)

    // Earth group
    const earthGroup = new THREE.Group()
    earthGroup.add(cloudsMesh)
    earthGroup.add(earthMesh)
    scene.add(earthGroup)

    // Camera
    const camera = new THREE.PerspectiveCamera(75)
    camera.position.z = 2.5
    camera.rotation.z = - Math.PI * 2 / 24
    scene.add(camera)

    // Lights
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.01)
    scene.add(ambientLight)

    const pointLight = new THREE.PointLight(0xffffee, 1.2, 0, .1)
    pointLight.position.set(-1, 1, 3)
    scene.add(pointLight)

    // Threejs loop
    const clock = new THREE.Clock()
    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        // Animation (for testing purpose)
        earthGroup.rotation.y = elapsedTime * .3
        cloudsMesh.rotation.y = elapsedTime * .005

        renderer.render(scene, camera)
        window.requestAnimationFrame(tick)
    }
    tick()

    // Resize handler
    const resizeHandler = () =>
    {
        let w = canvas.clientWidth
        let h = canvas.clientHeight
        camera.aspect = w / h
        camera.updateProjectionMatrix()
        // False to let the browser handle the canvas size
        renderer.setSize(w, h, false)
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    }
    resizeHandler()
    window.addEventListener('resize', resizeHandler)
})
</script>

<template>
    <canvas></canvas>
</template>

<style scoped>
    canvas {
        position: fixed;
        left: 0;
        top: 0;
        bottom: 0;
        right: 0;
        z-index: -1;
        width: 100vw;
        height: 100vh;
        object-fit: contain;
    }
</style>

