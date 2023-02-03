<script setup>
import { onMounted, onBeforeUpdate, ref } from 'vue'
import * as THREE from 'three'
import gsap from "gsap"
import { CustomEase } from "gsap/CustomEase"
gsap.registerPlugin(CustomEase)
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import * as dat from 'lil-gui'

const CONFIG = {
    debug: window && window.location.hash === '#debug',
    enableOrbit: false,
    maxCameraPan: .1,
    cameraMouseSpeed: .4,
    cameraZ: 2.3,
    ringTextureResolution: 2048 * 2 * 2,
    backgroundColor: 0x132b3a,
    earth: {
        radius: 1,
        // Texture can be offset from the 0 latitude/longitude
        textureOffset: Math.PI / 2,
    },
    atmosphere: {
        c: 1.08,
        p: 14,
        alpha: .6,
        scale: 1.08,
    },
    rings: {
        rotation: .16
    },
    ringTitle: {
        radius: 1.5,
        height: .22,
        y: .06,
        color: 0xffe4aa,
        rotationSpeed: .2,
    },
    ringDesc: {
        radius: 1.54,
        height: .06,
        y: -.06,
        color: 0xffcdcd,
        rotationSpeed: .06,
    },
    clouds: {
        color: 0xffe4af,
    },
    marker: {
        color: 0xffe4af,
        bottomWidth: .01,
        topWidth: .018,
        height: .5,
        opacity: .8,
    },
    visibility: {
        axesHelper: false,
        earth: true,
        clouds: true,
        atmosphere: true,
        ringTitle: true,
        ringDesc: true,
    },
    gps: {
        // And we want an additional offset because we dont want the GPS target to be at the screen *center* 
        latOffset: THREE.MathUtils.degToRad(-18),
        lngOffset: THREE.MathUtils.degToRad(-18)
    },
    transition: {
        earthRotationDuration: .6,
        cameraDuration: .8,
        cameraZ: .2,
        ringRotationDuration: .8,
        ringScaleValue: { x: 1.08, z: 1.08, y: .8 },
        ringScaleDuration: .8,
        markerDurationOut: .2,
        markerDurationIn: .3,
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
    // ===================================
    // Setup & Loading
    // ===================================

    // Load font
    var ringFont = new FontFace('ringFont', 'url(https://fonts.gstatic.com/s/jost/v14/92zPtBhPNqw79Ij1E865zBUv7mwKIjVBNIg.woff2)');
    await ringFont.load()
    document.fonts.add(ringFont);
    
    // Threejs setup
    const canvas = threeCanvas.value
    const scene = new THREE.Scene()
    scene.background = new THREE.Color(CONFIG.backgroundColor)
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


    // ===================================
    // Camera
    // ===================================

    const camera = new THREE.PerspectiveCamera(75)
    camera.position.z = CONFIG.cameraZ
    camera.rotation.z = - Math.PI * 2 / 24
    scene.add(camera)


    // ===================================
    // Earth (with clouds and atmosphere)
    // ===================================

    // Earth
    const earthMaterial = new THREE.MeshToonMaterial()
    earthMaterial.map = textureLoader.load('src/assets/earth3d/textures/earthmap.jpg')
    earthMaterial.metalnessMap = textureLoader.load('src/assets/earth3d/textures/earthmetalness.jpg')
    earthMaterial.roughnessMap = textureLoader.load('src/assets/earth3d/textures/earthroughness.jpg')
    earthMaterial.normalMap = textureLoader.load('src/assets/earth3d/textures/earthnormal.jpg')
    earthMaterial.normalScale = new THREE.Vector2(.9, .9)
    let gradientMap = textureLoader.load('src/assets/earth3d/textures/gradient.jpg')
    gradientMap.minFilter = THREE.NearestFilter
    gradientMap.magFilter = THREE.NearestFilter
    earthMaterial.gradientMap = gradientMap
    const earthMesh = new THREE.Mesh(new THREE.SphereGeometry(CONFIG.earth.radius, 32, 32), earthMaterial)
    earthMesh.rotation.y = -CONFIG.earth.textureOffset

    // Custom shader for earth atmosphere
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
    const earth = new THREE.Group()
    earth.add(cloudsMesh)
    earth.add(earthMesh)

    // Map Marker
    const markerMaterial = new THREE.MeshBasicMaterial({
        color: CONFIG.marker.color,
        transparent: true,
        opacity: CONFIG.marker.opacity,
        alphaMap: textureLoader.load('src/assets/earth3d/textures/gradient.png'),
    })
    const markerGeometry = new THREE.CylinderGeometry(CONFIG.marker.topWidth, CONFIG.marker.bottomWidth, CONFIG.marker.height, 8, 1, true)
    const markerMesh = new THREE.Mesh(markerGeometry, markerMaterial)
    markerMesh.position.set(0, 0, CONFIG.marker.height / 2)
    markerMesh.rotation.x = Math.PI / 2
    const marker = new THREE.Group()
    marker.add(markerMesh)
    marker.position.set(0, 0, CONFIG.earth.radius)

    // Earth group 2, to contain earth and marker
    // and to rotate the whole thing
    const earthGroup = new THREE.Group()
    earthGroup.rotation.set(CONFIG.gps.latOffset, CONFIG.gps.lngOffset, 0)
    earthGroup.add(earth)
    earthGroup.add(marker)
    scene.add(earthGroup)


    // ===================================
    // Rings
    // ===================================

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
    const ringTitle = new THREE.Group()
    ringTitle.add(ringTitleMesh)

    // Ring desc
    conf = CONFIG.ringDesc
    const ringDescGeometry = new THREE.CylinderGeometry(conf.radius, conf.radius, conf.height, 64, 1, true)
    const ringDescMaterial = new THREE.MeshToonMaterial({color: conf.color, transparent: true, alphaTest:.1})
    ringDescMaterial.side = THREE.DoubleSide
    ringDescMaterial.emissive = new THREE.Color(conf.color)
    ringDescMaterial.emissiveIntensity = .1
    const ringDescMesh = new THREE.Mesh(ringDescGeometry, ringDescMaterial)
    // To hold transition animations independently from ringDescMesh rotation
    const ringDesc = new THREE.Group()
    ringDesc.add(ringDescMesh)

    // Ring group
    const ringsGroup = new THREE.Group()
    ringsGroup.rotation.x = CONFIG.rings.rotation
    ringsGroup.add(ringTitle)
    ringsGroup.add(ringDesc)
    scene.add(ringsGroup)


    // ===================================
    // Lights
    // ===================================

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.01)
    scene.add(ambientLight)

    const pointLight = new THREE.PointLight(0xffffee, 1.2, 0, .1)
    pointLight.position.set(-1, 1, 3)
    scene.add(pointLight)


    // ===================================
    // Add references to objects into global scope
    // We do it at the end to chose which objects we want globally
    // ===================================

    threeObjects.camera = camera
    threeObjects.earth = earth
    threeObjects.marker = marker
    threeObjects.markerMaterial = markerMaterial
    threeObjects.ringTitleMaterial = ringTitleMaterial
    threeObjects.ringTitle = ringTitle
    threeObjects.ringDescMaterial = ringDescMaterial
    threeObjects.ringDesc = ringDesc
    threeObjects.ringsGroup = ringsGroup


    // ===================================
    // Force first update to show the right values
    // ===================================

    updateScene()


    // ===================================
    // Orbit controls (only available in debug mode)
    // ===================================

    const controls = new OrbitControls(camera, canvas)
    controls.enableDamping = true
    controls.enabled = false

    // ===================================
    // Axes helpers (only available in debug mode)
    // ===================================

    const axesHelper = new THREE.AxesHelper(2)
    axesHelper.visible = CONFIG.visibility.axesHelper
    scene.add(axesHelper)


    // ===================================
    // Threejs loop
    // ===================================

    const clock = new THREE.Clock()
    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        // Animation
        cloudsMesh.rotation.y = elapsedTime * .014

        // Rings params
        ringTitle.position.y = CONFIG.ringTitle.y
        ringTitleMesh.rotation.y = - elapsedTime * CONFIG.ringTitle.rotationSpeed
        ringTitleMaterial.opacity = .9 + Math.sin(elapsedTime * 2) * .1
        ringDesc.position.y = CONFIG.ringDesc.y
        ringDescMesh.rotation.y = - elapsedTime * CONFIG.ringDesc.rotationSpeed

        // Camera
        controls.enabled = CONFIG.enableOrbit
        if(!controls.enabled) {
            camera.lookAt(earth.position)
            camera.rotation.z = - Math.PI * 2 / 24
        } else {
            controls.update()
        }

        // Update atmo shader to always face camera
        atmoMaterial.uniforms.viewVector.value = new THREE.Vector3().subVectors( camera.position, atmoMesh.position )
        // Update atmo shader and mesh to adapt to CONFIG values in real time (for debug)
        atmoMaterial.uniforms.c.value = CONFIG.atmosphere.c
        atmoMaterial.uniforms.p.value = CONFIG.atmosphere.p
        atmoMaterial.uniforms.alpha.value = CONFIG.atmosphere.alpha
        atmoMesh.scale.set(CONFIG.atmosphere.scale, CONFIG.atmosphere.scale, CONFIG.atmosphere.scale)

        renderer.render(scene, camera)
        window.requestAnimationFrame(tick)
    }
    tick()


    // ===================================
    // Event: resize
    // ===================================

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


    // ===================================
    // Event: Mouse mouve
    // ===================================

    const mouseMoveHandler = (e) => {
        if(CONFIG.enableOrbit) {
            return
        }
        gsap.to(camera.position, {
            duration: CONFIG.cameraMouseSpeed,
            x: (- .5 + (e.clientX / window.innerWidth)) * CONFIG.maxCameraPan,
            y: (.5 - (e.clientY / window.innerHeight)) * CONFIG.maxCameraPan,
            ease: "power2.out"
        })
    }
    window.addEventListener('mousemove', mouseMoveHandler)


    // ===================================
    // Debug (init GUI)
    // ===================================
    if(CONFIG.debug) {
        console.log("Debug Mode!")
        const gui = new dat.GUI()

        let f = gui.addFolder("General")
        f.addColor(CONFIG, 'backgroundColor').onChange(() => {
            scene.background.set(CONFIG.backgroundColor)
        })
        f.add(CONFIG.visibility, "axesHelper").onChange(() => {            
            axesHelper.visible = CONFIG.visibility.axesHelper
        })
        f.add(CONFIG.visibility, "earth").onChange(() => {
            earthMesh.visible = CONFIG.visibility.earth
        })
        f.add(CONFIG.visibility, "clouds").onChange(() => {            
            cloudsMesh.visible = CONFIG.visibility.clouds
        })
        f.add(CONFIG.visibility, "atmosphere").onChange(() => {            
            atmoMesh.visible = CONFIG.visibility.atmosphere
        })
        f.add(CONFIG.visibility, "ringTitle").onChange(() => {            
            ringTitleMesh.visible = CONFIG.visibility.ringTitle
        })
        f.add(CONFIG.visibility, "ringDesc").onChange(() => {            
            ringDescMesh.visible = CONFIG.visibility.ringDesc
        })
        f.add(CONFIG.marker, "opacity", 0, 1, .01).name("marker opacity").onChange(() => {            
            markerMaterial.opacity = CONFIG.marker.opacity
        })
        f.addColor(CONFIG.marker, "color").name("marker color").onChange(() => {            
            markerMaterial.color.set(CONFIG.marker.color)
        })

        f = gui.addFolder('Camera')
        f.add(CONFIG, "enableOrbit")
        f.add(CONFIG, "maxCameraPan", 0, 4, .01)

        f = gui.addFolder('Atmosphere & Clouds')
        f.add(CONFIG.atmosphere, "c", 0, 3, .001)
        f.add(CONFIG.atmosphere, "p", 0, 30, .01)
        f.add(CONFIG.atmosphere, "alpha", 0, 1, .01)
        f.add(CONFIG.atmosphere, "scale", 1, 2, .01)
        f.addColor(CONFIG.clouds, 'color').name("clouds color").onChange(() => {
            cloudsMaterial.color.set(CONFIG.clouds.color)
            cloudsMaterial.needsUpdate = true
        })

        f = gui.addFolder('Ring (title)')
        f.add(CONFIG.ringTitle, "rotationSpeed", -5, 5, .001)
        f.add(CONFIG.ringTitle, "y", -5, 5, .01)
        f.addColor(CONFIG.ringTitle, 'color').onChange(() => {
            ringTitleMaterial.color.set(CONFIG.ringTitle.color)
        })

        f = gui.addFolder('Ring (description)')
        f.add(CONFIG.ringDesc, "rotationSpeed", -5, 5, .001)
        f.add(CONFIG.ringDesc, "y", -5, 5, .01)
        f.addColor(CONFIG.ringDesc, 'color').onChange(() => {
            ringDescMaterial.color.set(CONFIG.ringDesc.color)
        })
        
    }
}


// ===================================
// Update scene when changing props
// ===================================

function updateScene() {
    console.log("updateScene")
    // Update ring textures
    let title = props.title.toUpperCase() + " " + props.year
    threeObjects.ringTitleMaterial.map = getRingTexture(title, CONFIG.ringTitle)
    let desc = props.places.join(" - ").toUpperCase()
    threeObjects.ringDescMaterial.map = getRingTexture(desc, CONFIG.ringDesc)

    // Move earth so that gps position faces camera (only the earth moves)
    let duration = CONFIG.transition.earthRotationDuration
    let targetX = THREE.MathUtils.degToRad(props.lat) // + CONFIG.gps.latitudeTextureOffset
    let targetY = THREE.MathUtils.degToRad(-props.long) // + CONFIG.gps.longitudeTextureOffset
    gsap.to(threeObjects.earth.rotation, {
        duration: duration * 2,
        x: targetX,
        y: targetY,
        ease: "power2.inOut",
    })

    // Animate marker
    let markerTl = gsap.timeline()
    markerTl.to(threeObjects.markerMaterial, {
        duration: CONFIG.transition.markerDurationOut,
        opacity: 0,
        ease: "power1.inOut",
    }).to(threeObjects.marker.scale, {
        duration: CONFIG.transition.markerDurationOut,
        x: .2,
        y: .2,
        z: 1.2,
        ease: "power1.inOut",
    }, "<"
    ).to(threeObjects.markerMaterial, {
        duration: CONFIG.transition.markerDurationIn,
        opacity: CONFIG.marker.opacity,
        ease: "power1.inOut",
        delay: CONFIG.transition.earthRotationDuration / 2,
    }).to(threeObjects.marker.scale, {
        duration: CONFIG.transition.markerDurationIn,
        x: 1,
        y: 1,
        z: 1,
        ease: "power1.inOut",
    }, "<")


    duration = CONFIG.transition.markerDurationIn

    // Move camera during transition
    let cameraTl = gsap.timeline()
    duration = CONFIG.transition.cameraDuration
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
    gsap.fromTo(threeObjects.ringTitle.rotation, {
        y: 0,
    }, {
        duration: duration,
        y: -Math.PI * 2 * 1,
        ease: "power3.out",
    })
    gsap.fromTo(threeObjects.ringDesc.rotation, {
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


// ===================================
// Helper to get ring texture from a text
// ===================================

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

