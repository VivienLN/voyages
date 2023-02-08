<script setup>
import { onMounted, onBeforeUpdate, ref } from 'vue'
import * as THREE from 'three'
import { Line2 } from 'three/addons/lines/Line2.js';
import { LineMaterial } from 'three/addons/lines/LineMaterial.js';
import { LineGeometry } from 'three/addons/lines/LineGeometry.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import gsap from "gsap"
import { CustomEase } from "gsap/CustomEase"
gsap.registerPlugin(CustomEase)
import * as dat from 'lil-gui'

const props = defineProps({
    title: {
        type: String,
        required: true
    },
    previous: {
        type: Object,
        required: true
    },
    next: {
        type: Object,
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

const CONFIG = {
    debug: window && window.location.hash === '#debug',
    enableOrbit: false,
    cameraMouseSpeed: .4,
    backgroundColor: 0x132b3a,
    camera: {
        z: 2.3,
        raycastPlaneZ: 2,
        roll: - Math.PI * 2 / 24,
        maxPan: .2,
    },
    arrows: {
        z: 0,
        size: 1,
        offset: {
            x: .11,
            y: .2,
        },
        textureResolution: 512,
        alpha: .3,
        alphaHover: 1,
        scaleHover: 1.1,
        transition: 1.2,
        // Sizes are relative to the texture resolution
        fontSize: .05, 
        maskHeight: .07,
        textHeight: .12,
        lineWidth: .016,
        lineLength: .6,
    },
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
        rotation: .16,
        textureResolution: 2048 * 2 * 2,
    },
    ringTitle: {
        radius: 1.5,
        height: .2,
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
        cameraDurationOut: .4,
        cameraDurationIn: .6,
        cameraZ: .14,
        ringRotationDuration: .8,
        ringScaleValue: { x: 1.08, z: 1.08, y: .8 },
        ringScaleDuration: .8,
        markerDurationOut: .2,
        markerDurationIn: .3,
        arrowTextDuration: .4,
        arrowDelay: .1,
    }
}

// Holds values that are updated by gsap
// and used by some frame-by-frame animations
const animated = {
    arrows: {
        text1Offset: 0,
        text2Offset: 0,
        textAlpha: 1,
        leftText: "",
        rightText: "",
    },
}

// ThreeJS global variables/const
const threeCanvas = ref(null)
const threeObjects = {
    camera: null,
    leftArrow: null,
    rightArrow: null,
    earth: null,
    marker: null,
    markerMaterial: null,
    ringTitleMaterial: null,
    ringTitle: null,
    ringDescMaterial: null,
    ringDesc: null,
    ringsGroup: null,
    leftArrowCanvas : null,
    rightArrowCanvas : null,
}

// Mounted: init
onMounted(initScene)

// Update : update scene according to props
onBeforeUpdate(updateScene)

async function initScene() {
    // ===================================
    // Setup & Loading
    // ===================================

    // For mouse events
    var hovered = null
    const mouseRaycaster = new THREE.Raycaster()

    // Load font
    let font = new FontFace('font', 'url(https://fonts.gstatic.com/s/quicksand/v30/6xK-dSZaM9iE8KbpRA_LJ3z8mH9BOJvgkP8o58a-wg.woff2)');
    await font.load()
    document.fonts.add(font);
    let fontBold = new FontFace('fontBold', 'url(https://fonts.gstatic.com/s/jost/v14/92zPtBhPNqw79Ij1E865zBUv7mwKIjVBNIg.woff2)');
    await fontBold.load()
    document.fonts.add(fontBold);
    
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
    camera.position.z = CONFIG.camera.z
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
    const marker = new THREE.Mesh(markerGeometry, markerMaterial)
    marker.position.set(0, 0, CONFIG.marker.height / 2)
    marker.rotation.x = Math.PI / 2
    const markerGroup = new THREE.Group()
    markerGroup.add(marker)
    markerGroup.position.set(0, 0, CONFIG.earth.radius)

    // Earth group 2, to contain earth and marker
    // and to rotate the whole thing
    const earthGroup = new THREE.Group()
    earthGroup.rotation.set(CONFIG.gps.latOffset, CONFIG.gps.lngOffset, 0)
    earthGroup.add(earth)
    earthGroup.add(markerGroup)
    scene.add(earthGroup)

    // Marker line (from marker on surface to mouse position)
    const markerLineGeometry = new LineGeometry()
    const markerLine = new Line2(markerLineGeometry, new LineMaterial({
        color: 0xffffff,
        linewidth: .003,
        dashed: false,
        alphaToCoverage: false,
        transparent: true,
        depthWrite: false
    }))
    scene.add(markerLine)


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
    // Arrows
    // ===================================

    let arrowSize = CONFIG.arrows.size
    
    const leftArrowCanvas = document.createElement('canvas')
    let leftTexture = new THREE.Texture(leftArrowCanvas)
    leftTexture.minFilter = THREE.LinearFilter
    leftTexture.magFilter = THREE.NearestFilter
    const leftArrow = new THREE.Mesh(
        new THREE.PlaneGeometry(arrowSize, arrowSize), 
        new THREE.MeshBasicMaterial({transparent: true, opacity: CONFIG.arrows.alpha, map: leftTexture})
    )
    leftArrow.rotation.z = CONFIG.camera.roll

    
    const rightArrowCanvas = document.createElement('canvas')
    let rightTexture = new THREE.Texture(rightArrowCanvas)
    rightTexture.minFilter = THREE.LinearFilter
    rightTexture.magFilter = THREE.NearestFilter
    const rightArrow = new THREE.Mesh(
        new THREE.PlaneGeometry(arrowSize, arrowSize), 
        new THREE.MeshBasicMaterial({transparent: true, opacity: CONFIG.arrows.alpha, map: rightTexture})
    )
    rightArrow.rotation.z = CONFIG.camera.roll

    const arrows = new THREE.Group()
    arrows.add(leftArrow)
    arrows.add(rightArrow)
    scene.add(arrows)


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
    threeObjects.leftArrow = leftArrow
    threeObjects.rightArrow = rightArrow
    threeObjects.earth = earth
    threeObjects.markerGroup = markerGroup
    threeObjects.marker = marker
    threeObjects.ringTitleMaterial = ringTitleMaterial
    threeObjects.ringTitle = ringTitle
    threeObjects.ringDescMaterial = ringDescMaterial
    threeObjects.ringDesc = ringDesc
    threeObjects.ringsGroup = ringsGroup
    threeObjects.leftArrowCanvas = leftArrowCanvas
    threeObjects.rightArrowCanvas = rightArrowCanvas


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
            camera.rotation.z = CONFIG.camera.roll
        } else {
            controls.update()
        }

        // Update arrows distance to camera
        arrows.position.z = camera.position.z - CONFIG.camera.z 

        // Update arrows textures
        refreshArrowTexture(
            leftArrowCanvas, 
            true
        )
        refreshArrowTexture(
            rightArrowCanvas, 
            false
        )
        leftArrow.material.map.needsUpdate = true
        rightArrow.material.map.needsUpdate = true


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

        // Arrows
        let raycaster = new THREE.Raycaster()
        raycaster.setFromCamera({
            x: -(1 - CONFIG.arrows.offset.x),
            y: 1 - CONFIG.arrows.offset.y,
        }, camera)
        
        let plane = new THREE.Plane().setFromNormalAndCoplanarPoint(
            new THREE.Vector3(0, 0, 1), 
            new THREE.Vector3(0, 0, CONFIG.arrows.z)
        )
        let leftArrowPosition = new THREE.Vector3()
        raycaster.ray.intersectPlane(plane, leftArrowPosition)
        leftArrowPosition.add(new THREE.Vector3(CONFIG.arrows.size / 2, -CONFIG.arrows.size / 2, 0))
        leftArrow.position.copy(leftArrowPosition)
        rightArrow.position.set(
            -leftArrowPosition.x,
            -leftArrowPosition.y,
            leftArrowPosition.z
        )
        
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

        // Mouse position (between -1 and 1) & raycaster
        let position = {
            x: (e.clientX / window.innerWidth) * 2 - 1,
            y: - (e.clientY / window.innerHeight) * 2 + 1
        }
        
        mouseRaycaster.setFromCamera( position, camera );

        // Arrows
        let intersects = mouseRaycaster.intersectObjects(arrows.children);
        hovered = null
        arrows.children.forEach((arrow) => {
            let hover = intersects.find((intersect) => intersect.object === arrow)
            let targetOpacity = hover ? CONFIG.arrows.alphaHover : CONFIG.arrows.alpha
            let targetScale = hover ? CONFIG.arrows.scaleHover : 1
            gsap.to(arrow.material, {
                duration: CONFIG.arrows.transition,
                opacity: targetOpacity,
                ease: "power4.out"
            })
            gsap.to(arrow.scale, {
                duration: CONFIG.arrows.transition,
                x: targetScale,
                y: targetScale,
                ease: "power4.out"
            })
            if(hover) {
                hovered = arrow
            }
        })
        document.body.style.cursor = intersects.length ? "pointer" : "default";

        // Line from marker to mouse cursor
        let lineFrom = new THREE.Vector3()
        markerGroup.getWorldPosition(lineFrom)
        let lineTo = new THREE.Vector3()
        let plane = new THREE.Plane().setFromNormalAndCoplanarPoint(
            new THREE.Vector3(0, 0, 1), 
            new THREE.Vector3(0, 0, CONFIG.camera.raycastPlaneZ)
        )
        mouseRaycaster.ray.intersectPlane(plane, lineTo)

        let line = new THREE.Line3(lineFrom, lineTo)

        let offsetStart = new THREE.Vector3()
        line.at(0.2, offsetStart)

        markerLineGeometry.setPositions([
            offsetStart.x, offsetStart.y, offsetStart.z,
            line.end.x, line.end.y, line.end.z
        ])

        markerLine.material.opacity = line.distance() > 1.2 ? .2 : 1

        console.log(line.distance())

        // Camera (disabled if orbit is enabled)
        if(CONFIG.enableOrbit) {
            return
        }

        gsap.to(camera.position, {
            duration: CONFIG.cameraMouseSpeed,
            x: (position.x / 2) * CONFIG.camera.maxPan,
            y: (position.y / 2) * CONFIG.camera.maxPan,
            ease: "power2.out"
        })
    }
    window.addEventListener('mousemove', mouseMoveHandler)


    // ===================================
    // Event: Mouse click
    // ===================================

    const mouseClickHandler = (e) => {
        if(hovered === leftArrow) {
            canvas.dispatchEvent(new Event('onClickPrevious'))
        } else if(hovered === rightArrow) {
            canvas.dispatchEvent(new Event('onClickNext'))
        }
    }
    window.addEventListener('click', mouseClickHandler)


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
        f.add(CONFIG.camera, "maxPan", 0, 4, .01)

        f = gui.addFolder('Arrows')
        f.add(CONFIG.arrows, "size", 0, 2, .001).onChange(() => {
            let arrowSize = CONFIG.arrows.size
            leftArrow.geometry.dispose()
            rightArrow.geometry.dispose()
            arrows.remove(leftArrow)
            arrows.remove(rightArrow)
            leftArrow = new THREE.Mesh(new THREE.PlaneGeometry(arrowSize, arrowSize), new THREE.MeshBasicMaterial({
                color: 0xffffff,
                transparent: true,
            }))
            rightArrow = new THREE.Mesh(new THREE.PlaneGeometry(arrowSize, arrowSize), new THREE.MeshBasicMaterial({
                color: 0xffff00,
                transparent: true,
            }))

            leftArrow.rotation.z = CONFIG.camera.roll
            rightArrow.rotation.z = CONFIG.camera.roll

            arrows.add(leftArrow)
            arrows.add(rightArrow)
            resizeHandler()
        })
        f.add(CONFIG.arrows, "z", -3, 3, .01).onChange(resizeHandler)
        f.add(CONFIG.arrows.offset, "x", -.5, .5, .01).name("offset x").onChange(resizeHandler)
        f.add(CONFIG.arrows.offset, "y", -.5, .5, .01).name("offset y").onChange(resizeHandler)
        f.add(CONFIG.arrows, "alpha", 0, 1, .01)
        f.add(CONFIG.arrows, "alphaHover", 0, 1, .01)
        f.add(CONFIG.arrows, "scaleHover", 0, 3, .01)
        f.add(CONFIG.arrows, "transition", 0, 10, .01)

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

    // Update ring textures
    let title = props.title.toUpperCase() + " " + props.year
    threeObjects.ringTitleMaterial.map = getRingTexture(title, CONFIG.ringTitle)
    let desc = props.places.join(" - ").toUpperCase()
    threeObjects.ringDescMaterial.map = getRingTexture(desc, CONFIG.ringDesc)

    // Move earth so that gps position faces camera (only the earth moves)
    let duration = CONFIG.transition.earthRotationDuration
    let targetX = THREE.MathUtils.degToRad(props.lat)
    let targetY = THREE.MathUtils.degToRad(-props.long)
    gsap.to(threeObjects.earth.rotation, {
        duration: duration * 2,
        x: targetX,
        y: targetY,
        ease: "power2.inOut",
    })

    // Animate arrows text
    let arrowTl = gsap.timeline()
    gsap.killTweensOf(animated.arrows)
    arrowTl.to(animated.arrows, {
        textAlpha: 0,
        text1Offset: 1,
        duration: CONFIG.transition.arrowTextDuration,
        ease: "exp.in",
        onComplete: () => {
            animated.arrows.rightText = props.next.title.toUpperCase()
            animated.arrows.leftText = props.previous.title.toUpperCase()
        }
    }).to(animated.arrows, {
        delay: CONFIG.transition.arrowDelay,
        text2Offset: 1,
        duration: CONFIG.transition.arrowTextDuration,
        ease: "exp.in",
    }, "<"
    ).to(animated.arrows, {
        textAlpha: 1,
        text1Offset: 0,
        duration:  CONFIG.transition.arrowTextDuration,
        ease: "exp.out"
    }).to(animated.arrows, {
        delay: CONFIG.transition.arrowDelay,
        text2Offset: 0,
        duration: CONFIG.transition.arrowTextDuration,
        ease: "exp.out",
    }, "<"
    )

    // Animate marker
    let markerTl = gsap.timeline()
    markerTl.to(threeObjects.marker.material, {
        duration: CONFIG.transition.markerDurationOut,
        opacity: 0,
        ease: "power1.inOut",
    }).to(threeObjects.markerGroup.scale, {
        duration: CONFIG.transition.markerDurationOut,
        x: .2,
        y: .2,
        z: 1.2,
        ease: "power1.inOut",
    }, "<"
    ).to(threeObjects.marker.material, {
        duration: CONFIG.transition.markerDurationIn,
        opacity: CONFIG.marker.opacity,
        ease: "power1.inOut",
        delay: CONFIG.transition.earthRotationDuration / 2,
    }).to(threeObjects.markerGroup.scale, {
        duration: CONFIG.transition.markerDurationIn,
        x: 1,
        y: 1,
        z: 1,
        ease: "power1.inOut",
    }, "<")

    // Move camera during transition
    let cameraTl = gsap.timeline()
    duration = CONFIG.transition.cameraDuration
    cameraTl
        .to(threeObjects.camera.position, {
            duration: CONFIG.transition.cameraDurationOut,
            z: CONFIG.camera.z + CONFIG.transition.cameraZ,
            ease: "sine.inOut",
        }).to(threeObjects.camera.position, {
            duration: CONFIG.transition.cameraDurationIn,
            z: CONFIG.camera.z,
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
// Helper to get ring & arrow texture from text
// ===================================

// Called only on scene update
function getRingTexture(text, params) {
    // Create canvas
    let textMargin = 80
    let textureWidth = CONFIG.rings.textureResolution
    let textureRatio = Math.PI * 2 * params.radius / params.height
    let canvas = document.createElement('canvas')
    canvas.width = textureWidth
    canvas.height = canvas.width / textureRatio

    // Init context
    let ctx = canvas.getContext('2d')
    ctx.textBaseline = 'middle'
    ctx.font = `${canvas.height}px fontBold`

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

// Called frame by frame by threejs main loop (tick() function)
function refreshArrowTexture(canvas, isLeft) { 
    // Create canvas & context
    let textureSize = CONFIG.arrows.textureResolution
    canvas.width = textureSize
    canvas.height = textureSize
    let ctx = canvas.getContext('2d')
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    // Text
    let fontSize = CONFIG.arrows.fontSize * textureSize
    let textHeight = CONFIG.arrows.textHeight * textureSize
    ctx.textBaseline = 'middle'
    ctx.fillStyle = 'white'
    
    // For animation
    let maskHeight = CONFIG.arrows.maskHeight * textureSize
    let text1Offset = maskHeight * animated.arrows.text1Offset
    let text2Offset = maskHeight * animated.arrows.text2Offset
    ctx.globalAlpha = animated.arrows.textAlpha

    if(isLeft) {
        let text = animated.arrows.leftText
        let textY = textHeight / 2
        let prefix = "Précédent // ".toUpperCase()
        ctx.textAlign = "left"
        ctx.font = `${fontSize}px font`
        ctx.fillText(prefix, 0, textY + text1Offset)
        
        let textX = ctx.measureText(prefix).width
        ctx.font = `${fontSize}px fontBold`
        ctx.fillText(text, textX, textY + text2Offset)
        
        // Mask
        ctx.globalCompositeOperation = 'destination-in'
        ctx.fillRect(0, (textHeight - maskHeight) / 2, textureSize, maskHeight)
        ctx.globalCompositeOperation = 'source-over'

    } else {
        let text = animated.arrows.rightText
        let textY = textureSize - (textHeight / 2)
        ctx.textAlign = "right"

        ctx.font = `${fontSize}px fontBold`
        ctx.fillText(text, textureSize, textY - text1Offset)
        
        let prefix = "Suivant // ".toUpperCase()
        let prefixX = textureSize - ctx.measureText(text).width
        ctx.font = `${fontSize}px font`
        ctx.fillText(prefix, prefixX, textY - text2Offset)
        
        // Mask
        ctx.globalCompositeOperation = 'destination-in'
        ctx.fillRect(0, textureSize - (textHeight + maskHeight) / 2, textureSize, maskHeight)
        ctx.globalCompositeOperation = 'source-over'
    }

    ctx.globalAlpha = 1

    // Arrow lines
    let lineLength = CONFIG.arrows.lineLength * textureSize
    let lineWidth = CONFIG.arrows.lineWidth * textureSize
    let points = [
        { x: 0, y: lineLength },
        { x: (isLeft ? 0 : lineLength), y: (isLeft ? 0 : lineLength) },
        { x: lineLength, y: 0 },
    ]
    let offset = {
        x: isLeft ? (lineWidth / 2) : (textureSize - lineLength - (lineWidth / 2)),
        y: isLeft ? textHeight : (textureSize - lineLength - textHeight),
    }
    ctx.lineWidth = lineWidth
    ctx.strokeStyle = 'white'
    ctx.lineCap = "round"
    ctx.beginPath()
    points.forEach((p, i) => {
        (i == 0) ? ctx.moveTo(offset.x + p.x, offset.y + p.y) : ctx.lineTo(offset.x + p.x, offset.y + p.y)
    })
    ctx.stroke()
}

</script>

<template>
    <canvas 
        ref="threeCanvas"
        @onClickPrevious="$emit('clickPrevious')"
        @onClickNext="$emit('clickNext')"
    ></canvas>
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

