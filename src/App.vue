<script setup>
import { ref, computed, onMounted, nextTick, reactive } from 'vue'
import { gsap } from "gsap"
// 定义 photobox ref 对象，用于管理相册的交互和布局
const photobox = ref({
  container: null, // 相册容器 DOM 元素
  img_data: [], // 图像数据数组，存储每个图像的位置和移动信息
  container_width: 0, // 容器宽度
  container_height: 0, // 容器高度
  photo_width: 0, // 单个照片宽度（动态计算）
  photo_height: 0, // 单个照片高度（动态计算）
  if_movable: false, // 是否可拖动
  mouse_x: 0, // 鼠标 X 坐标
  mouse_y: 0, // 鼠标 Y 坐标
  standard_width: 1440, // 标准宽度，用于缩放计算
  scale_nums: 1, // 缩放比例
  autoSpeed: -100, // 自动滑动速度（像素/秒，向左）
  // 初始化方法，设置事件监听和初始布局
  init() {
    this.resize()
    window.addEventListener("resize", this.resize.bind(this))
    this.container.addEventListener("mousedown", this.handleMouseDown.bind(this))
    this.container.addEventListener("mouseup", this.handleMouseUp.bind(this))
    this.container.addEventListener("mouseleave", this.handleMouseLeave.bind(this))
    this.container.addEventListener("mousemove", this.handleMouseMove.bind(this))
    gsap.ticker.add(this.autoMove.bind(this))
  },
  // 读取图像文件，返回图像数据数组
  readImage() {
    const modules = import.meta.glob('./assets/images/*.{jpg,jpeg,png,gif,svg}')
    const filenames = Object.keys(modules).map(key => key.split('/').pop())
    console.log("纯图像文件名数组:", filenames)
    return Object.keys(modules).map((key, i) => ({
      id: i + 1,
      src: new URL(key.slice(2), import.meta.url).href
    }))
  },
  // 调整布局大小，重置图像位置和缩放
  resize() {
    const imgs = [...this.container.querySelectorAll(".photos_line_photo")]
    if (imgs.length === 0) return
    this.container_width = this.container.offsetWidth
    this.container_height = this.container.offsetHeight
    if (imgs[0]) {
      this.photo_width = imgs[0].offsetWidth
      this.photo_height = imgs[0].offsetHeight
    }
    this.scale_nums = document.body.offsetWidth / this.standard_width
    this.container.style.transform = `translate(-50%, -50%) scale(${this.scale_nums})`
    gsap.to(imgs, {
      transform: `translate(0,0)`,
      duration: 0,
      ease: 'power4.out'
    })
    this.img_data = imgs.map(img => ({
      node: img,
      x: img.offsetLeft, // 初始左偏移
      y: img.offsetTop, // 初始上偏移
      width: img.offsetWidth,
      height: img.offsetHeight,
      mov_x: 0, // X 移动偏移
      mov_y: 0, // Y 移动偏移
      ani: null // 动画实例
    }))
  },
  // 鼠标按下事件，启用拖动
  handleMouseDown(event) {
    this.if_movable = true
    this.mouse_x = event.clientX
    this.mouse_y = event.clientY
  },
  // 鼠标抬起事件，禁用拖动
  handleMouseUp() {
    this.if_movable = false
  },
  // 鼠标离开容器事件，禁用拖动
  handleMouseLeave() {
    this.if_movable = false
  },
  // 鼠标移动事件，处理拖动逻辑和无限滚动
  handleMouseMove(event) {
    if (!this.if_movable || !this.img_data) return
    const x = event.clientX
    const y = event.clientY
    const distance_x = (x - this.mouse_x) / this.scale_nums
    const distance_y = (y - this.mouse_y) / this.scale_nums
    this.img_data.forEach((img) => {
      let duration = 1
      img.mov_x += distance_x
      // X 方向无限滚动检查
      if (img.x + img.mov_x > this.container_width) {
        img.mov_x -= this.container_width
        duration = 0
      }
      if (img.x + img.mov_x < -img.width) {
        img.mov_x += this.container_width
        duration = 0
      }
      img.mov_y += distance_y
      // Y 方向无限滚动检查
      if (img.y + img.mov_y > this.container_height) {
        img.mov_y -= this.container_height
        duration = 0
      }
      if (img.y + img.mov_y < -img.height) {
        img.mov_y += this.container_height
        duration = 0
      }
      if (img.ani) img.ani.kill()
      img.ani = gsap.to(img.node, {
        transform: `translate(${img.mov_x}px,${img.mov_y}px)`,
        duration: duration,
        ease: 'power4.out'
      })
    })
    this.mouse_x = x
    this.mouse_y = y
  },
  // 自动滑动方法
  autoMove() {
    if (this.if_movable || !this.img_data.length) return
    const deltaTime = gsap.ticker.deltaRatio(60) * 16.666 // 近似帧时间 (ms)
    const delta_x = this.autoSpeed * (deltaTime / 1000) / this.scale_nums
    this.img_data.forEach((img) => {
      img.mov_x += delta_x
      if (img.x + img.mov_x < -img.width) {
        img.mov_x += this.container_width
      } else if (img.x + img.mov_x > this.container_width) {
        img.mov_x -= this.container_width
      }
      gsap.set(img.node, {
        transform: `translate(${img.mov_x}px, ${img.mov_y}px)`
      })
    })
  }
})
// 配置对象
const config = {
  imagesPerRow: 6, // 每行图像数量
  rowHeight: 150, // 行高
  spacing: 0, // 图像间距
  totalImages: 0 // 总图像数
}
const images = ref([]) // 图像列表
const imageSizes = ref({}) // 图像尺寸信息
const showPreview = ref(false) // 是否显示预览
const selectedImage = ref(null) // 选中的图像
// 加载图像尺寸
const loadImageSize = (image) => {
  return new Promise((resolve) => {
    const img = new Image()
    img.onload = () => {
      imageSizes.value[image.id] = {
        width: img.width,
        height: img.height,
        ratio: img.width / img.height
      }
      resolve(imageSizes.value[image.id])
    }
    img.onerror = () => {
      imageSizes.value[image.id] = {
        width: 200,
        height: 150,
        ratio: 4/3
      }
      resolve(imageSizes.value[image.id])
    }
    img.src = image.src
  })
}
// 计算布局，按行调整宽度以填充容器，修复初始位置和间距问题
const calculateLayout = () => {
  const container = document.querySelector(".photos")
  if (!container) return
  const containerWidth = container.offsetWidth || window.innerWidth
  images.value.forEach(image => {
    const sizeInfo = imageSizes.value[image.id]
    const ratio = sizeInfo ? sizeInfo.ratio : 4/3
    image.naturalWidth = ratio * config.rowHeight
    image.aspectRatio = ratio
  })
  const numRows = rows.value
  for (let rowIndex = 0; rowIndex < numRows; rowIndex++) {
    const start = rowIndex * config.imagesPerRow
    const end = Math.min(start + config.imagesPerRow, images.value.length)
    const rowImages = images.value.slice(start, end)
    const numInRow = rowImages.length
    if (numInRow === 0) continue
    const availableWidth = containerWidth - config.spacing * (numInRow - 1)
    let sumNatural = 0
    rowImages.forEach(img => {
      sumNatural += img.naturalWidth
    })
    let scale = 1
    if (sumNatural > 0) {
      scale = availableWidth / sumNatural
    }
    rowImages.forEach(img => {
      img.displayWidth = img.naturalWidth * scale
      img.displayHeight = config.rowHeight
    })
  }
  config.totalImages = images.value.length
}
// 计算行数
const rows = computed(() => Math.ceil(images.value.length / config.imagesPerRow))
// 打开预览
const openPreview = (image) => {
  selectedImage.value = image
  showPreview.value = true
}
// 关闭预览
const closePreview = () => {
  showPreview.value = false
  selectedImage.value = null
}
function startTunnel() {
  const canvas = document.getElementById('tunnel')
  const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl')
  if (!gl) return
  const resize = () => {
    canvas.width = innerWidth
    canvas.height = innerHeight
    gl.viewport(0, 0, innerWidth, innerHeight)
  }
  window.addEventListener('resize', resize)
  resize()
  const vert = `
    attribute vec2 a;
    void main() {
      gl_Position = vec4(a, 0.0, 1.0);
    }
  `
  const frag = `
    precision highp float;
    uniform float time;
    uniform vec2 res;
    vec3 hue(float t) {
      return .5 + .5 * cos(6.3 * t + vec3(0, 2, 4));
    }
    float rnd(vec2 p) {
      return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
    }
    void main() {
      vec2 uv = (gl_FragCoord.xy - .5 * res.xy) / res.y;
      float t = time * .2;
      vec3 col = vec3(0);
      for (float i = 0.; i < 3.; i++) {
        float z = fract(t - i * .3);
        float size = mix(20., .5, z);
        vec2 p = uv * size;
        p += vec2(sin(t * 1.5 + i), cos(t * 1.1 + i));
        float r = length(p);
        float stars = smoothstep(.1, .08, rnd(floor(p * 60.)) * r);
        col += hue(i * .3 + rnd(vec2(i))) * stars * (1. - z);
      }
      gl_FragColor = vec4(col, 1.);
    }
  `
  function createShader(type, src) {
    const s = gl.createShader(type)
    gl.shaderSource(s, src)
    gl.compileShader(s)
    return s
  }
  const prog = gl.createProgram()
  gl.attachShader(prog, createShader(gl.VERTEX_SHADER, vert))
  gl.attachShader(prog, createShader(gl.FRAGMENT_SHADER, frag))
  gl.linkProgram(prog)
  gl.useProgram(prog)
  const buf = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buf)
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]), gl.STATIC_DRAW)
  const locA = gl.getAttribLocation(prog, 'a')
  const locTime = gl.getUniformLocation(prog, 'time')
  const locRes = gl.getUniformLocation(prog, 'res')
  gl.enableVertexAttribArray(locA)
  gl.vertexAttribPointer(locA, 2, gl.FLOAT, false, 0, 0)
  let start = performance.now()
  function loop() {
    const t = (performance.now() - start) / 1000
    gl.uniform1f(locTime, t)
    gl.uniform2f(locRes, canvas.width, canvas.height)
    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)
    requestAnimationFrame(loop)
  }
  loop()
}
// 组件挂载时执行
onMounted(async () => {
  startTunnel()
  images.value = photobox.value.readImage().map(reactive)
  await Promise.all(images.value.map(loadImageSize))
  await nextTick()
  calculateLayout()
  await nextTick()
  photobox.value.container = document.querySelector(".photos")
  if (photobox.value.container) {
    photobox.value.init()
  }
  window.addEventListener('resize', async () => {
    calculateLayout()
    await nextTick()
    photobox.value.resize()
  })
})
</script>
<template>
  <div class="stage">
    <!-- 星轨隧道画布 -->
    <canvas id="tunnel"></canvas>
    
    <!-- 旧电影噪点+扫描线 -->
    <div class="grain"></div>
    
    <!-- 相册容器 -->
    <div class="photos">
      <div 
        class="photos_line" 
        v-for="(row, rowIndex) in Array.from({ length: rows })" 
        :key="rowIndex"
      >
        <div 
          class="photos_line_photo" 
          v-for="image in images.slice(rowIndex * config.imagesPerRow, (rowIndex + 1) * config.imagesPerRow)" 
          :key="image.id"
          :style="{
            width: `${image.displayWidth}px`,
            height: `${image.displayHeight}px`,
            margin: `0 ${config.spacing / 2}px`
          }"
          @click="openPreview(image)"
        >
          <img :src="image.src" :alt="`Photo ${image.id}`">
        </div>
      </div>
    </div>
    
    <!-- 图片预览模态框 -->
    <div v-if="showPreview" class="preview-modal" @click="closePreview">
      <img :src="selectedImage.src" alt="Preview" class="preview-image">
    </div>
  </div>
</template>
<style scoped>
/* 相册样式 */
.photos {
  width: 100%;
  overflow: hidden;
  box-sizing: border-box;
}
/* 行样式 */
.photos_line {
  display: flex;
  white-space: nowrap;
  align-items: flex-start;
  justify-content: flex-start;
  margin-bottom: 5px;
}
/* 单个照片样式 */
.photos_line_photo {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s ease;
  cursor: pointer;
}
/* hover 效果 */
.photos_line_photo:hover {
  transform: scale(1.05);
  z-index: 1;
}
/* 图像样式 */
.photos_line_photo img {
  display: block;
  object-fit: cover;
  width: 100%;
  height: 100%;
}
/* 预览模态样式 */
.preview-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
/* 预览图像样式 */
.preview-image {
  max-width: 90%;
  max-height: 90%;
  object-fit: contain;
}
/*====== 炫酷背景 ======*/
.stage {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000;
}
#tunnel {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}
.grain {
  position: absolute;
  inset: 0;
  z-index: 3;
  pointer-events: none;
  background: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='200' height='200'%3E%3Cfilter id='n'%3E%3CfeTurbulence baseFrequency='.8' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='.15'/%3E%3C/svg%3E");
  animation: scan 4s linear infinite;
}
@keyframes scan {
  0% { transform: translateY(-100%); }
  100% { transform: translateY(100%); }
}


/* 把相册变成悬浮毛玻璃卡片 */
.photos {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 90vw;
  height: 80vh;
  overflow: hidden;
  background: rgba(255, 255, 255, 0.1); /* 改为透明玻璃背景 */
  backdrop-filter: none; /* 移除毛玻璃模糊效果 */
  -webkit-backdrop-filter: none;
  border: 1px solid rgba(255, 255, 255, 0.3); /* 加强边框透明度 */
  border-radius: 16px;
  box-shadow: 
    0 0 30px rgba(0, 255, 255, 0.4), 
    0 0 60px rgba(255, 0, 255, 0.3),
    inset 0 0 20px rgba(255, 255, 255, 0.1); /* 添加内发光增强玻璃感 */
  z-index: 2;
  padding: 20px 0 0 20px;
  box-sizing: border-box;
}

</style>