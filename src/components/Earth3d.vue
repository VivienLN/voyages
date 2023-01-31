<script setup>
import { onMounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const CONFIG = {
    enableOrbit: false,
    maxCameraPan: .5,
    cameraZ: 2.8,
    gps: {
        latitude: 48.85655595428851,
        longitude: 2.352021668887112, 
        // Texture can be offset from the 0 latitude/longitude
        // And we want an additional offset because we dont want the GPS target to be at the *center*
        latitudeOffset: 0 - 18,
        longitudeOffset: -90 - 12,
    }
}

onMounted(async () => {
    // Load font
    var ringFont = new FontFace('ringFont', 'url(https://fonts.gstatic.com/s/jost/v14/92zPtBhPNqw79Ij1E865zBUv7mwKIjVBNIg.woff2)');
    await ringFont.load()
    document.fonts.add(ringFont);
    
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
    // earthMaterial.normalScale = new THREE.Vector2(.1, .1)
    earthMaterial.normalScale = new THREE.Vector2(.9, .9)
    // earthMaterial.emissiveMap = textureLoader.load('src/assets/earth3d/textures/earthmetalness.jpg')
    // earthMaterial.emissive = new THREE.Color(0x222222)
    let gradientMap = textureLoader.load('src/assets/earth3d/textures/gradient.jpg')
    gradientMap.minFilter = THREE.NearestFilter
    gradientMap.magFilter = THREE.NearestFilter
    earthMaterial.gradientMap = gradientMap
    const earthMesh = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), earthMaterial)

    // Clouds
    const cloudsTexture = textureLoader.load('src/assets/earth3d/textures/clouds.jpg')
    const cloudsMaterial = new THREE.MeshToonMaterial({color: 0xffe4af, transparent: true, opacity: .9})
    cloudsMaterial.alphaMap = cloudsTexture
    cloudsMaterial.metalness = 0
    cloudsMaterial.roughness = .6
    cloudsMaterial.gradientMap = gradientMap
    const cloudsMesh = new THREE.Mesh(new THREE.SphereGeometry(1.02, 32, 32), cloudsMaterial)

    // Ring
    const ringParams = {
        radius: 1.6,
        height: .3,
    }
    const ringTexture = getRingTexture("Lorem ipsum".toUpperCase(), ringParams)
    const ringGeometry = new THREE.CylinderGeometry(ringParams.radius, ringParams.radius, ringParams.height, 64, 1, true)
    const ringMaterial = new THREE.MeshToonMaterial({map: ringTexture, color: 0xffffff, transparent: true, opacity: .9})
    ringMaterial.side = THREE.DoubleSide
    const ringMesh = new THREE.Mesh(ringGeometry, ringMaterial)
    ringMesh.rotation.x = .09
    scene.add(ringMesh)

    // Earth group
    const earthGroup = new THREE.Group()
    earthGroup.add(cloudsMesh)
    earthGroup.add(earthMesh)
    scene.add(earthGroup)

    // Move earth so that gps position faces camera (only the earth moves)
    // earthGroup.rotation.x = CONFIG.gps.latitude * Math.PI / 180
    earthGroup.rotation.x = (CONFIG.gps.latitude + CONFIG.gps.latitudeOffset) * Math.PI / 180
    earthGroup.rotation.y = (-CONFIG.gps.longitude + CONFIG.gps.longitudeOffset) * Math.PI / 180

    // Camera
    const camera = new THREE.PerspectiveCamera(75)
    camera.position.z = CONFIG.cameraZ
    camera.rotation.z = - Math.PI * 2 / 24
    scene.add(camera)

    // Lights
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.01)
    scene.add(ambientLight)

    const pointLight = new THREE.PointLight(0xffffee, 1.2, 0, .1)
    pointLight.position.set(-1, 1, 3)
    scene.add(pointLight)

    // Controls (for debugging)
    const controls = CONFIG.enableOrbit ? new OrbitControls(camera, canvas) : null
    if(controls) {
        controls.enableDamping = true
    }

    // Threejs loop
    const clock = new THREE.Clock()
    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        // Animation (for testing purpose)
        // earthGroup.rotation.y = elapsedTime * .3
        cloudsMesh.rotation.y = elapsedTime * .01
        ringMesh.rotation.y = - elapsedTime * .1
        camera.lookAt(earthGroup.position)
        camera.rotation.z = - Math.PI * 2 / 24
            
        // Update controls (for testing)
        if(controls) {
            controls.update()
        }

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

    // Mouse mouve handler
    const mouseMoveHandler = (e) => 
    {
        camera.position.x = (e.clientX / window.innerWidth) / 2 * CONFIG.maxCameraPan
        camera.position.y = - (e.clientY / window.innerHeight) / 2 * CONFIG.maxCameraPan
    }
    window.addEventListener('mousemove', mouseMoveHandler)
})

function getRingTexture(text, params) {
    // Create canvas
    let textMargin = 60
    let textureWidth = 2048 * 2
    let textureRatio = Math.PI * 2 * params.radius / params.height
    let canvas = document.createElement('canvas')
    canvas.width = textureWidth
    canvas.height = canvas.width / textureRatio

    // Init context
    let ctx = canvas.getContext('2d')
    ctx.textBaseline = 'middle'
    ctx.font = `${canvas.height}px ringFont`

    // Calculate repetitions
    let textWidth = ctx.measureText(text).width + textMargin
    let repetitions = Math.floor(textureWidth / textWidth)

    // Draw text
    for(let i = 0; i < repetitions; i++) {
        let x = canvas.width * i / repetitions
        ctx.fillStyle = 'white'
        ctx.fillText(text, x, canvas.height / 2)
    }

    // Create texture from canvas
    let texture = new THREE.Texture(canvas)
    texture.needsUpdate = true

    return texture 
}

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
        z-index: 2;
        width: 100vw;
        height: 100vh;
        object-fit: contain;
    }
</style>

