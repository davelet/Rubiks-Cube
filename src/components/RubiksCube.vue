<template>
  <div ref="container"></div>
</template>

<script lang="ts" setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import anime from 'animejs'
import { Pane } from 'tweakpane'

const rotateAroundWorldAxis = (origin: THREE.Vector3, vector: THREE.Vector3, radius: number) => {
  vector.normalize()
  const u = vector.x
  const v = vector.y
  const w = vector.z
  const a = origin.x
  const b = origin.y
  const c = origin.z
  const matrix4 = new THREE.Matrix4()
  matrix4.set(
    u * u + (v * v + w * w) * Math.cos(radius), u * v * (1 - Math.cos(radius)) - w * Math.sin(radius), u * w * (1 - Math.cos(radius)) + v * Math.sin(radius), (a * (v * v + w * w) - u * (b * v + c * w)) * (1 - Math.cos(radius)) + (b * w - c * v) * Math.sin(radius),
    u * v * (1 - Math.cos(radius)) + w * Math.sin(radius), v * v + (u * u + w * w) * Math.cos(radius), v * w * (1 - Math.cos(radius)) - u * Math.sin(radius), (b * (u * u + w * w) - v * (a * u + c * w)) * (1 - Math.cos(radius)) + (c * u - a * w) * Math.sin(radius),
    u * w * (1 - Math.cos(radius)) - v * Math.sin(radius), v * w * (1 - Math.cos(radius)) + u * Math.sin(radius), w * w + (u * u + v * v) * Math.cos(radius), (c * (u * u + v * v) - w * (a * u + b * v)) * (1 - Math.cos(radius)) + (a * v - b * u) * Math.sin(radius),
    0, 0, 0, 1
  )
  return matrix4
}

enum AxesEnum { x, y, z }
const XYZ_VALUE = 0 ^ 1 ^ 2

const AxesVec3 = {
  'x+': new THREE.Vector3(1, 0, 0),
  'x-': new THREE.Vector3(-1, 0, 0),
  'y+': new THREE.Vector3(0, 1, 0),
  'y-': new THREE.Vector3(0, -1, 0),
  'z+': new THREE.Vector3(0, 0, 1),
  'z-': new THREE.Vector3(0, 0, -1)
}

const roundRectByArc = (ctx: CanvasRenderingContext2D, ...[x, y, w, h, r]: number[]) => {
  const min = Math.min(w, h), PI = Math.PI
  if (r > min / 2) r = min / 2
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.lineTo(x + r, y)
  ctx.lineTo(x + w - r, y)
  ctx.arc(x + w - r, y + r, r, -PI / 2, 0)
  ctx.lineTo(x + w, y + h - r)
  ctx.arc(x + w - r, y + h - r, r, 0, PI / 2)
  ctx.lineTo(x + r, y + h)
  ctx.arc(x + r, y + h - r, r, PI / 2, PI)
  ctx.lineTo(x, y + r)
  ctx.arc(x + r, y + r, r, PI, -PI / 2)
  ctx.closePath()
}

const colors = [
  '#3b81f5', // 蓝色
  '#029c56', // 绿色
  '#d94335', // 红色
  '#e66f00', // 橘色
  '#f3b30a', // 黄色
  '#f4f4f4', // 白色
]

const getFaceColor = (
  color: string,
  radius = 10,
  gutter = 5,
  gutterColor = '#000',
  text = ''
): HTMLCanvasElement => {
  const size = 256
  const canvas = document.createElement('canvas')
  canvas.width = canvas.height = size
  const ctx = canvas.getContext('2d')!
  ctx.fillStyle = gutterColor
  ctx.fillRect(0, 0, size, size)
  ctx.fillStyle = color
  roundRectByArc(ctx, gutter, gutter, size - gutter * 2, size - gutter * 2, radius)
  ctx.fill()
  if (text) {
    ctx.fillStyle = gutterColor
    const h = size / 2
    ctx.font = `${h}px Microsoft YaHei`
    ctx.textBaseline = 'top'
    const w = ctx.measureText(text).width
    ctx.fillText(text, size / 2 - w / 2, size / 2 - h / 2)
  }
  return canvas

}


// const pane: any = new Pane()
const debug = {
  level: 3,
  rotateTime: 300,
  gutter: 5,
  radius: 20,
  gutterColor: '#222',
  cubeSize: 2,
}

class CreepCube {
  width: number
  height: number
  renderer!: THREE.WebGLRenderer
  scene!: THREE.Scene
  camera!: THREE.PerspectiveCamera
  controls!: OrbitControls
  intersect1: THREE.Intersection | null = null
  intersect2: THREE.Intersection | null = null
  startPlane: THREE.Vector3 | undefined = undefined
  startPoint: THREE.Vector3 | null = null
  movePoint: THREE.Vector3 | null = null
  touchCube: THREE.Mesh | null = null
  isRotating = false
  isShuffling = false
  isSolving = false
  mouse = new THREE.Vector2(0, 0)
  raycaster = new THREE.Raycaster()
  mesh!: THREE.Group
  children: THREE.Mesh[] = []
  coordinate: THREE.Vector3[] = []
  private pane: Pane
  constructor(config = { level: 3 }) {
    this.pane = new Pane()
    this.width = window.innerWidth
    this.height = window.innerHeight
    this.initRenderer()
    this.initScene()
    this.initCamera()
    this.initControls()
    this.initDebug()
    this.initCube()
    this.render()

    window.addEventListener('mousedown', this.onMouseDown.bind(this))
    window.addEventListener('touchstart', this.onMouseDown.bind(this))
    window.addEventListener('mousemove', this.onMouseMove.bind(this))
    window.addEventListener('touchmove', this.onMouseMove.bind(this))
    window.addEventListener('mouseup', this.onMouseUp.bind(this))
    window.addEventListener('touchend', this.onMouseUp.bind(this))
  }

  // 从 Base 类移过来的方法
  initRenderer() {
    const renderer = this.renderer = new THREE.WebGLRenderer({
      antialias: true,
      alpha: true
    })
    renderer.setSize(this.width, this.height)
    renderer.setPixelRatio(window.devicePixelRatio)
    document.body.appendChild(renderer.domElement)
  }

  initScene() {
    this.scene = new THREE.Scene()
  }

  initCamera() {
    const camera = this.camera = new THREE.PerspectiveCamera(45, this.width / this.height, 0.1, 100)
    camera.position.set(15, 15, 15)
    camera.lookAt(0, 0, 0)
  }

  initControls() {
    const controls = this.controls = new OrbitControls(this.camera, this.renderer.domElement)
    controls.target = new THREE.Vector3(0, 0, 0)
    controls.enableZoom = false
    controls.rotateSpeed = 1
    controls.update()
  }

  render() {
    this.controls.update()
    this.renderer.clear()
    this.renderer.render(this.scene, this.camera)
    window.requestAnimationFrame(() => {
      this.render()
    })
  }

  computeMaterial(
    rules: boolean[],
    materials: THREE.MeshBasicMaterial[],
    defaultMaterial: THREE.MeshBasicMaterial,
    logoMaterial?: THREE.MeshBasicMaterial,
  ) {
    return rules.map((item, index) => item ? logoMaterial || materials[index] : defaultMaterial)
  }

  async initCube() {
    this.mesh && this.scene.remove(this.mesh)
    this.children = []
    this.coordinate = []
    const mesh = this.mesh = new THREE.Group()
    this.scene.add(mesh)
    // 小立方体
    const { level, gutter, radius, cubeSize: size } = debug
    const w = level, h = level, d = level
    const offsetWidth = w * size / 2 - size / 2
    const offsetHeight = h * size / 2 - size / 2
    const offsetDepth = d * size / 2 - size / 2
    const geometry = new THREE.BoxGeometry(size, size, size)
    // 材质
    const materials = colors.map(color => {
      const canvas = getFaceColor(color, radius, gutter, debug.gutterColor)
      const texture = new THREE.Texture(canvas)
      texture.needsUpdate = true
      texture.minFilter = THREE.NearestFilter
      texture.generateMipmaps = false
      return new THREE.MeshBasicMaterial({ map: texture })
    })
    const defaultMaterial = new THREE.MeshBasicMaterial({ color: debug.gutterColor })
    for (let i = 0; i < d; i++) {
      for (let j = 0; j < w * h; j++) {
        const index = i * w * h + j
        const currMaterial = this.computeMaterial([
          (j + 1) % w === 0,
          j % w === 0,
          i === d - 1,
          i === 0,
          j + w >= w * h,
          j < w,
        ], materials, defaultMaterial)
        const mesh = new THREE.Mesh(geometry, currMaterial)
        mesh.position.set(
          (j % w) * size - offsetWidth,
          size * i - offsetDepth,
          (j / w >> 0) * size - offsetHeight,
        )
        mesh.name = String(index)
        this.mesh.add(mesh)
        this.children.push(mesh)
        this.coordinate[index] = mesh.position.clone()
      }
    }
    // 外壳
    const outside = new THREE.Mesh(
      new THREE.BoxGeometry(
        size * w + 0.01,
        size * d + 0.01,
        size * h + 0.01
      ),
      new THREE.MeshBasicMaterial({
        color: '#fff',
        opacity: 0,
        transparent: true
      })
    )
    mesh.add(outside)
    // // 自动打乱
    // this.shuffleCube()
  }

  // 旋转逻辑
  computeRotation() {
    const sub = this.movePoint?.sub(this.startPoint!)
    if (!sub) return
    let minAngle = Infinity, minIndex = 0, planeNormalIndex: number = NaN
    Object.values(AxesVec3).forEach((vec3, index) => {
      const angle = sub.angleTo(vec3)
      if (vec3.angleTo(this.startPlane!) === 0) {
        planeNormalIndex = index
      }
      if (angle < minAngle) {
        minAngle = angle
        minIndex = index
      }
    })
    const direction = planeNormalIndex % 2 === 0 ? 1 : -1
    const planeNormalKey = planeNormalIndex / 2 >> 0
    const planeNormal = AxesEnum[planeNormalKey]
    // console.log('平面法向量', planeNormal, planeNormalIndex)
    const rotationVector = ['x+', 'x-', 'y+', 'y-', 'z+', 'z-'][minIndex]
    // console.log('旋转向量', rotationVector)
    let rotationAxis
    if (planeNormal === 'x') {
      if (rotationVector === 'y-') rotationAxis = 'z+'
      if (rotationVector === 'y+') rotationAxis = 'z-'
      if (rotationVector === 'z+') rotationAxis = 'y+'
      if (rotationVector === 'z-') rotationAxis = 'y-'
    }
    if (planeNormal === 'y') {
      if (rotationVector === 'x+') rotationAxis = 'z+'
      if (rotationVector === 'x-') rotationAxis = 'z-'
      if (rotationVector === 'z+') rotationAxis = 'x-'
      if (rotationVector === 'z-') rotationAxis = 'x+'
    }
    if (planeNormal === 'z') {
      if (rotationVector === 'y-') rotationAxis = 'x-'
      if (rotationVector === 'y+') rotationAxis = 'x+'
      if (rotationVector === 'x-') rotationAxis = 'y+'
      if (rotationVector === 'x+') rotationAxis = 'y-'
    }
    if (!rotationAxis) return
    this.rotateCube(this.touchCube!, rotationAxis as keyof typeof AxesVec3, direction)
  }

  // 根据立方体索引反推层数
  getStoreyByIndex(axis: 'x' | 'y' | 'z', i: number) {
    const lev = debug.level
    const squa = lev ** 2
    const _k = lev - 1
    const rules = {
      x: (i: number) => _k - ((i % squa) % lev),
      y: (i: number) => _k - (i / squa >> 0),
      z: (i: number) => _k - ((i % squa) / lev >> 0)
    }
    return rules[axis](i)
  }

  // 根据层数获取立方体
  getCubesByStorey(axis: 'x' | 'y' | 'z', storey = 0, sort = 1) {
    const lev = debug.level
    const squa = lev ** 2
    const rules: {
      [key in typeof axis]: ((n: unknown, i: number) => boolean)[]
    } = {
      x: [],
      y: [],
      z: [],
    }
    for (let k = 0; k < lev; k++) {
      const _k = lev - k - 1
      rules['x'][k] = (n, i) => i % lev === _k
      rules['y'][k] = (n, i) => i / squa >> 0 === _k
      rules['z'][k] = (n, i) => (i % 9) / lev >> 0 === _k
    }
    storey = sort > 0 ? storey : lev - storey - 1
    const rule = rules[axis][storey]
    return this.children.filter(rule)
  }

  /**
   * 旋转逻辑
   */
  async rotateCube(touchCube: THREE.Mesh, rotationAxis: keyof typeof AxesVec3, direction: 1 | -1) {
    if (this.isRotating) return
    this.isRotating = true
    const vec3 = AxesVec3[rotationAxis]
    const axis = rotationAxis.slice(0, 1)
    if (!vec3) return this.clearState()
    const allPromise: Promise<void>[] = []
    this.children.forEach(item => {
      if (touchCube.position[axis] === item.position[axis]) {
        allPromise.push(this.moveCube(item, vec3, direction))
      }
    })
    await Promise.all(allPromise)
    this.clearState()
    this.resetCubes()
  }

  clearState() {
    this.isRotating = false
    this.startPoint = null
    this.movePoint = null
  }

  // 移动小方块
  moveCube(cube: THREE.Mesh, vec3: THREE.Vector3, direction: 1 | -1): Promise<void> {
    return new Promise(resolve => {
      let prev = 0, data = { progress: 0 }
      anime({
        targets: data,
        duration: debug.rotateTime,
        progress: 1,
        easing: 'easeOutQuad',
        update: () => {
          const interval = data.progress - prev
          prev = data.progress
          const matrix = rotateAroundWorldAxis(
            new THREE.Vector3(0, 0, 0),
            new THREE.Vector3(-vec3.x * direction, -vec3.y * direction, -vec3.z * direction),
            interval * Math.PI / 2
          )
          cube.applyMatrix4(matrix)
        },
        complete: () => {
          const p = cube.position
          cube.scale.set(1, 1, 1)
          cube.position.set(Math.round(p.x), Math.round(p.y), Math.round(p.z))
          resolve()
        }
      })
    })
  }

  resetCubes() {
    const map = {}
    const toString = (vec3: THREE.Vector3) => `x${vec3.x}y${vec3.y}z${vec3.z}`
    this.children.forEach(cube => map[toString(cube.position)] = cube)
    this.coordinate.forEach((vec3, index) => {
      this.children[index] = map[toString(vec3)]
    })
  }

  getMouseSite(e: MouseEvent | TouchEvent) {
    const x = (e['touches'] ? e['touches'][0] : e).clientX
    const y = (e['touches'] ? e['touches'][0] : e).clientY
    this.mouse.x = (x / this.width - 0.5) * 2
    this.mouse.y = -(y / this.height - 0.5) * 2
  }

  onMouseDown(e: MouseEvent | TouchEvent) {
    this.getMouseSite(e)
    this.raycaster.setFromCamera(this.mouse, this.camera)
    const intersects = this.raycaster.intersectObjects([this.mesh])
    this.controls.enabled = !intersects.length
    // 兼容移动端
    this.onMouseMove(e)
    // 旋转
    if (this.intersect1 && this.intersect2 && !this.isRotating) {
      // 起始点
      this.startPoint = this.intersect1.point
      // 起始平面
      this.startPlane = this.intersect1.face?.normal
      // 触摸的小方块
      this.touchCube = this.intersect2.object as THREE.Mesh
    }
  }

  onMouseMove(e: MouseEvent | TouchEvent) {
    if (!this.mesh) return
    this.getMouseSite(e)
    this.raycaster.setFromCamera(this.mouse, this.camera)
    const intersects = this.raycaster.intersectObjects(this.mesh.children)
    this.intersect1 = intersects[0]
    this.intersect2 = intersects[1]
    if (this.intersect1 && !this.isRotating && this.startPoint) {
      this.movePoint = this.intersect1.point
      if (this.movePoint.distanceTo(this.startPoint) > debug.cubeSize / 5) {
        this.computeRotation()
      }
    }
  }

  onMouseUp(e: MouseEvent | TouchEvent) {
    this.startPoint = null
    this.movePoint = null
    this.controls.enabled = true
  }

  async shuffleCube() {
    if (this.isShuffling) return
    this.isShuffling = true
    // this.controls.autoRotate = true
    this['inp1'].disabled = true
    this['btn1'].disabled = true
    this['btn2'].disabled = true
    debug.rotateTime = 150
    const count = debug.level * 5 + 10
    const children = this.children.slice()
    for (let i = 0; i < count; i++) {
      const axis = ['x', 'y', 'z'][i % 3]
      await this.rotateCube(
        children[Math.random() * children.length >> 0],
        `${axis}${Math.random() - 0.5 > 0 ? '+' : '-'}` as keyof typeof AxesVec3,
        Math.random() - 0.5 > 0 ? 1 : -1
      )
    }
    this.isShuffling = false
    // this.controls.autoRotate = false
    this['inp1'].disabled = false
    this['btn1'].disabled = false
    this['btn2'].disabled = false
    debug.rotateTime = 300
  }

  initDebug() {
    const f1 = this.pane.addFolder({ title: '参数' })
    this['inp1'] = f1.addBinding(debug, 'level', {
      label: '难度',
      options: {
        '二阶': 2,
        '三阶': 3,
        '四阶': 4,
        '五阶': 5,
        '六阶': 6,
        '七阶': 7,
      }
    }).on('change', (ev: any) => {
      const lev = ev.value
      this.initCube()
      this.camera.position.set(lev * 5, lev * 5, lev * 5)
      this.camera.lookAt(0, 0, 0)
    })
    f1.addBinding(debug, 'radius', {
      label: '圆角',
      min: 0,
      max: 100,
    }).on('change', () => {
      this.initCube()
    })
    f1.addBinding(debug, 'gutter', {
      label: '缝隙',
      min: 0,
      max: 50,
    }).on('change', () => {
      this.initCube()
    })
    f1.addBinding(debug, 'gutterColor', {
      label: '缝隙颜色',
    }).on('change', () => {
      this.initCube()
    })
    const f2 = this.pane.addFolder({ title: '操作' })
    this['btn1'] = f2.addButton({
      title: 'Reset 重置',
    }).on('click', () => {
      this.initCube()
    })
    this['btn2'] = f2.addButton({
      title: 'Shuffle 打乱',
    }).on('click', () => {
      this.shuffleCube()
    })
    this['btn3'] = f2.addButton({
      title: 'Solve 还原',
      disabled: true
    }).on('click', () => {
      this.shuffleCube()
    })
    // f2.addButton({ title: 'R' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'x+', 1)
    // })
    // f2.addButton({ title: 'R\'' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'x+', -1)
    // })
    // f2.addButton({ title: 'L' }).on('click', () => {
    //   this.rotateCube(this.children[24], 'x-', 1)
    // })
    // f2.addButton({ title: 'L\'' }).on('click', () => {
    //   this.rotateCube(this.children[24], 'x-', -1)
    // })
    // f2.addButton({ title: 'U' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'y+', 1)
    // })
    // f2.addButton({ title: 'U\'' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'y+', -1)
    // })
    // f2.addButton({ title: 'D' }).on('click', () => {
    //   this.rotateCube(this.children[8], 'y-', 1)
    // })
    // f2.addButton({ title: 'D\'' }).on('click', () => {
    //   this.rotateCube(this.children[8], 'y-', -1)
    // })
    // f2.addButton({ title: 'F' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'z+', 1)
    // })
    // f2.addButton({ title: 'F\'' }).on('click', () => {
    //   this.rotateCube(this.children[26], 'z+', -1)
    // })
    // f2.addButton({ title: 'B' }).on('click', () => {
    //   this.rotateCube(this.children[20], 'z-', 1)
    // })
    // f2.addButton({ title: 'B\'' }).on('click', () => {
    //   this.rotateCube(this.children[20], 'z-', -1)
    // })
    // f2.addButton({ title: 'x' }).on('click', async () => {
    //   if (this.isRotating) return
    //   this.isRotating = true
    //   const vec3 = AxesVec3['x+']
    //   const direction = 1
    //   const allPromise: Promise<void>[] = []
    //   this.children.forEach(item => {
    //     allPromise.push(this.moveCube(item, vec3, direction))
    //   })
    //   await Promise.all(allPromise)
    //   this.isRotating = false
    //   this.clearState()
    //   this.resetCubes()
    // })
  }
}
const container = ref<HTMLElement | null>(null)

onMounted(() => {
  if (container.value) {
    const cube = new CreepCube()
    container.value.appendChild(cube.renderer.domElement)
  }
})

onBeforeUnmount(() => {
  if (cube) {
    // 清理事件监听器
    window.removeEventListener('mousedown', cube.onMouseDown)
    window.removeEventListener('touchstart', cube.onMouseDown)
    window.removeEventListener('mousemove', cube.onMouseMove)
    window.removeEventListener('touchmove', cube.onMouseMove)
    window.removeEventListener('mouseup', cube.onMouseUp)
    window.removeEventListener('touchend', cube.onMouseUp)
    
    // 清理渲染器
    cube.renderer.dispose()
    cube = null
  }
})
</script>

<style scoped>
div {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>