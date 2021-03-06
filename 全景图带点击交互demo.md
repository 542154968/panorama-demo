# 代码

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8" />
        <title>将3D坐标的标记投影到屏幕上</title>
        <script type="text/javascript" src="libs/three.js"></script>
        <script type="text/javascript" src="libs/stats.min.js"></script>
        <script type="text/javascript" src="libs/dat.gui.min.js"></script>

        <script
            type="text/javascript"
            src="libs/controls/OrbitControls.js"
        ></script>

        <style>
            body {
                /* set margin to 0 and overflow to hidden, to go fullscreen */
                margin: 0;
                overflow: hidden;
            }
        </style>
    </head>
    <body>
        <div id="Stats-output"></div>
        <!-- Div which will hold the Output -->
        <div id="WebGL-output"></div>

        <script type="text/javascript">
            var scene = new THREE.Scene()

            var camera = new THREE.PerspectiveCamera(
                45,
                window.innerWidth / window.innerHeight,
                0.1,
                1000000
            )
            camera.position.set(0, 5, 33)
            camera.lookAt(scene.position)

            var renderer = new THREE.WebGLRenderer()
            renderer.setClearColor(0x000000, 1.0)
            renderer.setPixelRatio(window.devicePixelRatio)
            renderer.setSize(window.innerWidth, window.innerHeight)
            renderer.sortObjects = true

            renderer.shadowMap.enabled = true
            renderer.shadowMap.type = THREE.PCFShadowMap

            var ambientLight = new THREE.AmbientLight(0xffffff)
            scene.add(ambientLight)

            var light = new THREE.SpotLight(0xffffff, 1.1)
            light.position.set(0, 0, 0)
            light.castShadow = true
            scene.add(light)

            var tagObject = new THREE.Object3D()
            scene.add(tagObject)

            var controls = new THREE.OrbitControls(camera)

            document
                .getElementById('WebGL-output')
                .appendChild(renderer.domElement)

            var textureCube = createCubeMap()
            var shader = THREE.ShaderLib['cube']
            shader.uniforms['tCube'].value = textureCube
            var material = new THREE.ShaderMaterial({
                uniforms: shader.uniforms,
                vertexShader: shader.vertexShader,
                fragmentShader: shader.fragmentShader,
                depthWrite: false,
                side: THREE.BackSide
            })
            var geometry = new THREE.BoxGeometry(10000, 10000, 10000)
            var cubeMesh = new THREE.Mesh(geometry, material)

            // var loader = new THREE.TextureLoader(),
            //     texture = loader.load(
            //         'images/DJI_0114-output Panorama.jpg',
            //         function(obj) {
            //             renderer.render(scene, camera)
            //             return obj
            //         }
            //     ),
            //     cubeMesh = new THREE.Mesh(
            //         // 形状
            //         // new THREE.CubeGeometry(10, 10, 10),
            //         new THREE.SphereGeometry(50, 32, 32),
            //         // 材质
            //         new THREE.MeshPhongMaterial({
            //             // color: 0xff0000,
            //             side: THREE.DoubleSide,
            //             map: texture
            //         })
            //     )
            scene.add(cubeMesh)

            //添加射线代码
            var raycasterCubeMesh
            var raycaster = new THREE.Raycaster()
            var mouseVector = new THREE.Vector3()
            var tags = []

            document.addEventListener('mousemove', onMouseMove, false)
            document.addEventListener('mousedown', onMouseDown, false)

            render()

            var activePoint
            function onMouseMove(event) {
                mouseVector.x = 2 * (event.clientX / window.innerWidth) - 1
                mouseVector.y = -2 * (event.clientY / window.innerHeight) + 1

                raycaster.setFromCamera(mouseVector.clone(), camera)
                var intersects = raycaster.intersectObjects([cubeMesh])

                if (raycasterCubeMesh) {
                    scene.remove(raycasterCubeMesh)
                }
                activePoint = null
                if (intersects.length > 0) {
                    // var points = []
                    // points.push(new THREE.Vector3(0, 0, 0))
                    // points.push(intersects[0].point)

                    // var mat = new THREE.MeshBasicMaterial({
                    //     color: 0xff0000,
                    //     transparent: true,
                    //     opacity: 0.5
                    // })
                    // var sphereGeometry = new THREE.SphereGeometry(100)

                    // raycasterCubeMesh = new THREE.Mesh(sphereGeometry, mat)
                    // raycasterCubeMesh.position.copy(intersects[0].point)
                    // scene.add(raycasterCubeMesh)
                    activePoint = intersects[0].point
                }
            }
            function onMouseDown(event) {
                if (event.buttons === 2 && activePoint) {
                    var tagMesh = new THREE.Mesh(
                        new THREE.SphereGeometry(0.1),
                        new THREE.MeshBasicMaterial({ color: 0xffff00 })
                    )
                    tagMesh.position.copy(activePoint)
                    tagObject.add(tagMesh)

                    var tagElement = document.createElement('div')
                    tagElement.innerHTML =
                        '<span>标记' + (tags.length + 1) + '</span>'
                    tagElement.style.background = '#00ff00'
                    tagElement.style.position = 'absolute'
                    tagElement.addEventListener('click', function(evt) {
                        alert(tagElement.innerText)
                    })
                    tagMesh.updateTag = function() {
                        if (isOffScreen(tagMesh, camera)) {
                            tagElement.style.display = 'none'
                        } else {
                            tagElement.style.display = 'block'
                            var position = toScreenPosition(tagMesh, camera)
                            tagElement.style.left = position.x + 'px'
                            tagElement.style.top = position.y + 'px'
                        }
                    }
                    tagMesh.updateTag()
                    document
                        .getElementById('WebGL-output')
                        .appendChild(tagElement)
                    tags.push(tagMesh)
                }
            }

            function createCubeMap() {
                var path = 'assets/texture/cubemap/'
                var format = '.jpg'
                var context = ''

                var urls = [
                    path + 'posx' + context + format,
                    path + 'negx' + context + format,
                    path + 'posy' + context + format,
                    path + 'negy' + context + format,
                    path + 'posz' + context + format,
                    path + 'negz' + context + format
                ]

                var texture = THREE.ImageUtils.loadTextureCube(
                    urls,
                    THREE.CubeReflectionMapping
                )

                return texture
            }

            function toScreenPosition(obj, camera) {
                var vector = new THREE.Vector3()
                var widthHalf = 0.5 * renderer.context.canvas.width
                var heightHalf = 0.5 * renderer.context.canvas.height

                obj.updateMatrixWorld()
                vector.setFromMatrixPosition(obj.matrixWorld)
                vector.project(camera)

                vector.x = vector.x * widthHalf + widthHalf
                vector.y = -(vector.y * heightHalf) + heightHalf

                return {
                    x: vector.x,
                    y: vector.y
                }
            }

            function isOffScreen(obj, camera) {
                var frustum = new THREE.Frustum() //Frustum用来确定相机的可视区域
                var cameraViewProjectionMatrix = new THREE.Matrix4()
                cameraViewProjectionMatrix.multiplyMatrices(
                    camera.projectionMatrix,
                    camera.matrixWorldInverse
                ) //获取相机的法线
                frustum.setFromMatrix(cameraViewProjectionMatrix) //设置frustum沿着相机法线方向

                return !frustum.intersectsObject(obj)
            }

            function render() {
                controls.update()
                tags.forEach(function(tagMesh) {
                    tagMesh.updateTag()
                })
                renderer.render(scene, camera)
                requestAnimationFrame(render)
            }
        </script>
    </body>
</html>

```
