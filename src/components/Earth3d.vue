<script setup>
import { onMounted, onBeforeUpdate, ref } from 'vue'
import * as THREE from 'three'
import gsap from "gsap"
import { CustomEase } from "gsap/CustomEase"
gsap.registerPlugin(CustomEase)
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const CONFIG = {
    enableGui: false,
    enableOrbit: false,
    maxCameraPan: .1,
    cameraMouseSpeed: .4,
    cameraZ: 2.3,
    ringTextureResolution: 2048 * 2 * 2,
    atmosphere: {
        c: 1,
        p: 8,
        alpha: .5,
        scale: 1.06,
    },
    ringTitle: {
        radius: 1.5,
        height: .22,
        y: .06,
        color: 0xffe4aa,
    },
    ringDesc: {
        radius: 1.54,
        height: .06,
        y: -.06,
        color: 0xffcdcd,
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
    year: {
        type: Number,
        required: true
    },
    places: {
        type: Array,
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
    scene.background = new THREE.Color(0x132b3a)
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

    // Camera
    const camera = new THREE.PerspectiveCamera(75)
    camera.position.z = CONFIG.cameraZ
    camera.rotation.z = - Math.PI * 2 / 24
    scene.add(camera)

    // Earth
    const earthMaterial = new THREE.MeshToonMaterial()
    earthMaterial.map = textureLoader.load('src/assets/earth3d/textures/earthmap.jpg')
    earthMaterial.bumpMap = textureLoader.load('src/assets/earth3d/textures/earthbump.jpg')
    earthMaterial.bumpScale = .005
    earthMaterial.metalnessMap = textureLoader.load('src/assets/earth3d/textures/earthmetalness.jpg')
    earthMaterial.roughnessMap = textureLoader.load('src/assets/earth3d/textures/earthroughness.jpg')
    earthMaterial.normalMap = textureLoader.load('src/assets/earth3d/textures/earthnormal.jpg')
    earthMaterial.normalScale = new THREE.Vector2(.9, .9)
    let gradientMap = textureLoader.load('src/assets/earth3d/textures/gradient.jpg')
    gradientMap.minFilter = THREE.NearestFilter
    gradientMap.magFilter = THREE.NearestFilter
    earthMaterial.gradientMap = gradientMap
    const earthMesh = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), earthMaterial)

    // Custom shader for earth glow
    // thanks to https://github.com/stemkoski/stemkoski.github.com/blob/master/Three.js/Shader-Glow.html
    const atmoMaterial = new THREE.ShaderMaterial({
	    uniforms: 
		{ 
			"c":   { type: "f", value: CONFIG.atmosphere.c },
			"p":   { type: "f", value: CONFIG.atmosphere.p },
			"alpha":   { type: "f", value: CONFIG.atmosphere.alpha },
			glowColor: { type: "c", value: new THREE.Color(0x7777ff) },
			viewVector: { type: "v3", value: camera.position }
		},
		vertexShader: `
            uniform vec3 viewVector;
            uniform float c;
            uniform float p;
            varying float intensity;
            void main() 
            {
                vec3 vNormal = normalize( normalMatrix * normal );
                vec3 vNormel = normalize( normalMatrix * viewVector );
                intensity = pow( c - dot(vNormal, vNormel), p );

                gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
            }
        `,
		fragmentShader: `
            uniform vec3 glowColor;
            uniform float alpha;
            varying float intensity;
            void main() 
            {
                vec3 glow = glowColor * intensity;
                gl_FragColor = vec4( glow, alpha );
            }
        `,
		side: THREE.BackSide,
		blending: THREE.AdditiveBlending,
		transparent: true,
        depthWrite: false
	});
    const atmoMesh = new THREE.Mesh(new THREE.SphereGeometry(1, 64, 64), atmoMaterial)
    atmoMesh.scale.multiplyScalar(CONFIG.atmosphere.scale)
    scene.add(atmoMesh)

    // Clouds
    const cloudsTexture = textureLoader.load('src/assets/earth3d/textures/clouds.jpg')
    const cloudsMaterial = new THREE.MeshToonMaterial({
        color: CONFIG.clouds.color, 
        transparent: true, 
        opacity: 1,
        depthWrite: false
    })
    cloudsMaterial.alphaMap = cloudsTexture
    cloudsMaterial.metalness = 0
    cloudsMaterial.roughness = .6
    cloudsMaterial.gradientMap = gradientMap
    const cloudsMesh = new THREE.Mesh(new THREE.SphereGeometry(1.02, 32, 32), cloudsMaterial)

    // Earth group
    const earthGroup = new THREE.Group()
    earthGroup.add(cloudsMesh)
    earthGroup.add(earthMesh)
    scene.add(earthGroup)

    // Ring title
    let conf = CONFIG.ringTitle
    const ringTitleGeometry = new THREE.CylinderGeometry(conf.radius, conf.radius, conf.height, 64, 1, true)
    // alphaTest>0 to avoid transparency issues
    const ringTitleMaterial = new THREE.MeshToonMaterial({color: conf.color, transparent: true, alphaTest:.1})
    ringTitleMaterial.side = THREE.DoubleSide
    ringTitleMaterial.emissive = new THREE.Color(conf.color)
    ringTitleMaterial.emissiveIntensity = .1
    const ringTitleMesh = new THREE.Mesh(ringTitleGeometry, ringTitleMaterial)
    // To hold transition animations independently from ringTitleMesh rotation
    const ringTitleGroup = new THREE.Group()
    ringTitleGroup.position.y = conf.y
    ringTitleGroup.add(ringTitleMesh)

    // Ring desc
    conf = CONFIG.ringDesc
    const ringDescGeometry = new THREE.CylinderGeometry(conf.radius, conf.radius, conf.height, 64, 1, true)
    const ringDescMaterial = new THREE.MeshToonMaterial({color: conf.color, transparent: true, alphaTest:.1})
    ringDescMaterial.side = THREE.DoubleSide
    ringDescMaterial.emissive = new THREE.Color(conf.color)
    ringDescMaterial.emissiveIntensity = .1
    const ringDescMesh = new THREE.Mesh(ringDescGeometry, ringDescMaterial)
    // To hold transition animations independently from ringDescMesh rotation
    const ringDescGroup = new THREE.Group()
    ringDescGroup.position.y = conf.y
    ringDescGroup.add(ringDescMesh)

    // Ring group
    const ringsGroup = new THREE.Group()
    ringsGroup.rotation.x = .16
    ringsGroup.add(ringTitleGroup)
    ringsGroup.add(ringDescGroup)
    scene.add(ringsGroup)

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
    threeObjects.earthGroup = earthGroup
    threeObjects.ringTitleMaterial = ringTitleMaterial
    threeObjects.ringTitleGroup = ringTitleGroup
    threeObjects.ringDescMaterial = ringDescMaterial
    threeObjects.ringDescGroup = ringDescGroup
    threeObjects.ringsGroup = ringsGroup

    // Force first update to show the right values
    updateScene()

    // Threejs loop
    const clock = new THREE.Clock()
    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        // Animation (for testing purpose)
        cloudsMesh.rotation.y = elapsedTime * .014
        ringTitleMesh.rotation.y = - elapsedTime * .2
        ringDescMesh.rotation.y = - elapsedTime * .06

        // Camera
        if(!CONFIG.enableOrbit) {
            camera.lookAt(earthGroup.position)
            camera.rotation.z = - Math.PI * 2 / 24
        } else {
            controls.update()
        }

        // Update atmo shader to always face camera
        // atmoMaterial.uniforms.viewVector.value = camera.position
        atmoMaterial.uniforms.viewVector.value = new THREE.Vector3().subVectors( camera.position, atmoMesh.position )

        // Flikering ring
        ringTitleMaterial.opacity = .9 + Math.sin(elapsedTime * 2) * .1

        renderer.render(scene, camera)
        window.requestAnimationFrame(tick)
    }
    tick()

    // Event: resize
    const resizeHandler = () => {
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
            x: (- .5 + (e.clientX / window.innerWidth)) * CONFIG.maxCameraPan,
            y: (.5 - (e.clientY / window.innerHeight)) * CONFIG.maxCameraPan,
            ease: "power2.out"
        })
    }
    if(!CONFIG.enableOrbit) {
        window.addEventListener('mousemove', mouseMoveHandler)
    }
}

function updateScene() {
    console.log("updateScene")
    // Update ring textures
    // let titleTexture = 
    let title = props.title.toUpperCase() + " " + props.year
    threeObjects.ringTitleMaterial.map = getRingTexture(title, CONFIG.ringTitle)
    let desc = props.places.join(" - ").toUpperCase()
    threeObjects.ringDescMaterial.map = getRingTexture(desc, CONFIG.ringDesc)

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

    // Move rings to hide transition
    duration = CONFIG.transition.ringRotationDuration
    gsap.fromTo(threeObjects.ringTitleGroup.rotation, {
        y: 0,
    }, {
        duration: duration,
        y: -Math.PI * 2 * 1,
        ease: "power3.out",
    })
    gsap.fromTo(threeObjects.ringDescGroup.rotation, {
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
        .to(threeObjects.ringsGroup.scale, {
            duration: duration / 3,
            x: CONFIG.transition.ringScaleValue.x,
            y: CONFIG.transition.ringScaleValue.y,
            z: CONFIG.transition.ringScaleValue.z,
            ease: "power2.out",
        }).to(threeObjects.ringsGroup.scale, {
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
    let textureWidth = CONFIG.ringTextureResolution
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
    // texture.anisotropy = 4
    texture.needsUpdate = true
    texture.minFilter = THREE.LinearFilter
    texture.magFilter = THREE.NearestFilter

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
        /* z-index: 0; */
        width: 100vw;
        height: 100vh;
        object-fit: contain;
    }
</style>

