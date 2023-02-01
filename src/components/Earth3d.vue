<script setup>
import { onMounted, onBeforeUpdate, ref } from 'vue'
import * as THREE from 'three'
import gsap from "gsap"
import { CustomEase } from "gsap/CustomEase"
gsap.registerPlugin(CustomEase)
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const CONFIG = {
    enableOrbit: false,
    maxCameraPan: .7,
    cameraMouseSpeed: .4,
    cameraZ: 2.4,
    ring: {
        radius: 1.6,
        height: .22,
        color: 0xffe4aa,
    },
    clouds: {
        color: 0xffe4af,
    },
    gps: {
        // Texture can be offset from the 0 latitude/longitude
        // And we want an additional offset because we dont want the GPS target to be at the *center*
        latitudeOffset: 0 - 18,
        longitudeOffset: -90 - 12,
    },
    transition: {
        earthRotationDuration: .6,
        cameraDuration: 1,
        cameraZ: .2,
        ringRotationDuration: .8,
        ringScaleValue: { x: 1.08, z: 1.08, y: .8 },
        ringScaleDuration: .8,
    }
}

const props = defineProps({
    title: {
        type: String,
        required: true
    },
    lat: {
        type: Number,
        required: true
    },
    long: {
        type: Number,
        required: true
    },
})

// ThreeJS global variables/const
const threeCanvas = ref(null)
const threeObjects = {}

// Mounted: init
onMounted(initScene)

// Update : update scene according to props
onBeforeUpdate(updateScene)

async function initScene() {
    // Load font
    var ringFont = new FontFace('ringFont', 'url(https://fonts.gstatic.com/s/jost/v14/92zPtBhPNqw79Ij1E865zBUv7mwKIjVBNIg.woff2)');
    await ringFont.load()
    document.fonts.add(ringFont);
    
    // Threejs setup
    const canvas = threeCanvas.value
    const scene = new THREE.Scene()
    const renderer = new THREE.WebGLRenderer({
        alpha: true,
        canvas: canvas
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
    const cloudsMaterial = new THREE.MeshToonMaterial({color: CONFIG.clouds.color, transparent: true, opacity: .9})
    cloudsMaterial.alphaMap = cloudsTexture
    cloudsMaterial.metalness = 0
    cloudsMaterial.roughness = .6
    cloudsMaterial.gradientMap = gradientMap
    const cloudsMesh = new THREE.Mesh(new THREE.SphereGeometry(1.02, 32, 32), cloudsMaterial)

    // Ring
    const ringGeometry = new THREE.CylinderGeometry(CONFIG.ring.radius, CONFIG.ring.radius, CONFIG.ring.height, 64, 1, true)
    const ringMaterial = new THREE.MeshToonMaterial({color: CONFIG.ring.color, transparent: true, opacity: .9})
    ringMaterial.side = THREE.DoubleSide
    ringMaterial.needsUpdate = true
    const ringMesh = new THREE.Mesh(ringGeometry, ringMaterial)
    // To hold transition animations independently from ringMesh rotation
    const ringGroup = new THREE.Group()
    ringGroup.rotation.x = .09
    ringGroup.add(ringMesh)
    scene.add(ringGroup)

    // Earth group
    const earthGroup = new THREE.Group()
    earthGroup.add(cloudsMesh)
    earthGroup.add(earthMesh)
    scene.add(earthGroup)

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

    // Add references to objects into global scope
    // We do it at the end to chose which objects we want globally
    threeObjects.camera = camera
    threeObjects.ringMaterial = ringMaterial
    threeObjects.ringGroup = ringGroup
    threeObjects.earthGroup = earthGroup

    // Force first update to show the right values
    updateScene()

    // Threejs loop
    const clock = new THREE.Clock()
    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        // Animation (for testing purpose)
        cloudsMesh.rotation.y = elapsedTime * .014
        ringMesh.rotation.y = - elapsedTime * .18
        camera.lookAt(earthGroup.position)
        camera.rotation.z = - Math.PI * 2 / 24

        // Flikering ring
        ringMaterial.opacity = .9 + Math.sin(elapsedTime * 2) * .1
            
        // Update controls (for testing)
        if(controls) {
            controls.update()
        }

        renderer.render(scene, camera)
        window.requestAnimationFrame(tick)
    }
    tick()

    // Event: resize
    const resizeHandler = () => {
        console.log("resize")
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

    // Event: Mouse mouve
    const mouseMoveHandler = (e) => 
    {
        gsap.to(camera.position, {
            duration: CONFIG.cameraMouseSpeed,
            x: (e.clientX / window.innerWidth) / 2 * CONFIG.maxCameraPan,
            y: - (e.clientY / window.innerHeight) / 2 * CONFIG.maxCameraPan,
            ease: "power2.out"
        })
    }
    window.addEventListener('mousemove', mouseMoveHandler)
}

function updateScene() {
    console.log("updateScene")
    // Update ring texture
    threeObjects.ringMaterial.map = getRingTexture(props.title.toUpperCase(), CONFIG.ring)
    threeObjects.ringMaterial.needsUpdate = true

    // Move earth so that gps position faces camera (only the earth moves)
    let duration = CONFIG.transition.earthRotationDuration
    let targetX = (props.lat + CONFIG.gps.latitudeOffset) * Math.PI / 180
    let targetY = (-props.long + CONFIG.gps.longitudeOffset) * Math.PI / 180
    gsap.to(threeObjects.earthGroup.rotation, { 
        duration: duration,
        x: targetX,
        y: targetY,
        ease: "power2.inOut",
    })

    // Move camera during transition
    let cameraTl = gsap.timeline();
    duration = .8
    cameraTl
        .to(threeObjects.camera.position, {
            duration: CONFIG.transition.cameraDuration / 3,
            z: CONFIG.cameraZ + CONFIG.transition.cameraZ,
            ease: "sine.inOut",
        }).to(threeObjects.camera.position, {
            duration: CONFIG.transition.cameraDuration / 3 * 2,
            z: CONFIG.cameraZ,
            ease: "sine.inOut",
        })

    // Move ring to hide transition
    duration = CONFIG.transition.ringRotationDuration
    gsap.fromTo(threeObjects.ringGroup.rotation, {
        y: 0,
    }, {
        duration: duration,
        y: -Math.PI * 2 * 1,
        ease: "power3.out",
    })
    // Deform ring
    let ringScaleTl = gsap.timeline();
    duration = CONFIG.transition.ringScaleDuration
    ringScaleTl
        .to(threeObjects.ringGroup.scale, {
            duration: duration / 3,
            x: CONFIG.transition.ringScaleValue.x,
            y: CONFIG.transition.ringScaleValue.y,
            z: CONFIG.transition.ringScaleValue.z,
            ease: "power2.out",
        }).to(threeObjects.ringGroup.scale, {
            duration: duration / 3 * 2,
            x: 1,
            y: 1,
            z: 1,
            ease: "power2.out",
        })
}

function getRingTexture(text, params) {
    // Create canvas
    let textMargin = 80
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
    <canvas ref="threeCanvas"></canvas>
</template>

<style scoped>
    canvas {
        position: fixed;
        left: 0;
        top: 0;
        bottom: 0;
        right: 0;
        z-index: -2;
        width: 100vw;
        height: 100vh;
        object-fit: contain;
    }
</style>

