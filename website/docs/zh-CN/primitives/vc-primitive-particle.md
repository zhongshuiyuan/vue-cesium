## VcPrimitiveParticle

加载粒子系统图元，相当于初始化一个 `Cesium.ParticleSystem` 实例。

### 基础用法

粒子系统图元组件的基础用法。

:::demo 使用 `vc-primitive-particle` 标签在三维球上添加烟花粒子效果。

```html
<el-row ref="viewerContainer" class="demo-viewer">
  <vc-viewer @ready="onViewerReady" shouldAnimate>
    <vc-primitive-particle
      :ref="setItemRef"
      v-for="(option, index) of options"
      :key="index"
      :image="option.image"
      :color="option.color"
      :startColor="option.startColor"
      :endColor="option.endColor"
      :particleLife="option.particleLife"
      :speed="option.speed"
      :imageSize="option.imageSize"
      :emissionRate="option.emissionRate"
      :emitter="option.emitter"
      :bursts="option.bursts"
      :lifetime="option.lifetime"
      :updateCallback="option.updateCallback"
      :modelMatrix="option.modelMatrix"
      :emitterModelMatrix="option.emitterModelMatrix"
      @click="onClicked"
      @complete="onComplete"
    >
    </vc-primitive-particle>
  </vc-viewer>
  <el-row class="demo-toolbar">
    <el-button type="danger" round @click="unload">销毁</el-button>
    <el-button type="danger" round @click="load">加载</el-button>
    <el-button type="danger" round @click="reload">重载</el-button>
  </el-row>
</el-row>

<script>
  export default {
    data() {
      return {
        particleCanvas: null,
        options: [],
        list: []
      }
    },
    methods: {
      setItemRef(el) {
        this.list.indexOf(el) === -1 && this.list.push(el)
      },
      onViewerReady({ Cesium, viewer }) {
        window.vm = this
        var scene = viewer.scene
        scene.debugShowFramesPerSecond = true
        Cesium.Math.setRandomNumberSeed(315)
        this.modelMatrix = Cesium.Transforms.eastNorthUpToFixedFrame(Cesium.Cartesian3.fromDegrees(108.59777, 40.03883))
        this.emitterInitialLocation = new Cesium.Cartesian3(0.0, 0.0, 100.0)
        this.minimumExplosionSize = 30.0
        this.maximumExplosionSize = 100.0
        this.particlePixelSize = new Cesium.Cartesian2(7.0, 7.0)
        var burstSize = 400.0
        this.lifetime = 5
        var numberOfFireworks = 20.0

        var xMin = -100.0
        var xMax = 100.0
        var yMin = -80.0
        var yMax = 100.0
        var zMin = -50.0
        var zMax = 50.0

        var colorOptions = [
          {
            minimumRed: 0.75,
            green: 0.0,
            minimumBlue: 0.8,
            alpha: 1.0
          },
          {
            red: 0.0,
            minimumGreen: 0.75,
            minimumBlue: 0.8,
            alpha: 1.0
          },
          {
            red: 0.0,
            green: 0.0,
            minimumBlue: 0.8,
            alpha: 1.0
          },
          {
            minimumRed: 0.75,
            minimumGreen: 0.75,
            blue: 0.0,
            alpha: 1.0
          }
        ]

        const options = []
        for (var i = 0; i < numberOfFireworks; ++i) {
          var x = Cesium.Math.randomBetween(xMin, xMax)
          var y = Cesium.Math.randomBetween(yMin, yMax)
          var z = Cesium.Math.randomBetween(zMin, zMax)
          var offset = new Cesium.Cartesian3(x, y, z)
          var color = Cesium.Color.fromRandom(colorOptions[i % colorOptions.length])
          console.log(color)

          var bursts = []
          for (var j = 0; j < 3; ++j) {
            bursts.push(
              new Cesium.ParticleBurst({
                time: Cesium.Math.nextRandomNumber() * this.lifetime,
                minimum: burstSize,
                maximum: burstSize
              })
            )
          }

          options.push(this.createFirework(offset, color, bursts))
        }
        this.options = options
        var camera = viewer.scene.camera
        var cameraOffset = new Cesium.Cartesian3(-300.0, 0.0, 0.0)
        camera.lookAtTransform(this.modelMatrix, cameraOffset)
        camera.lookAtTransform(Cesium.Matrix4.IDENTITY)

        var toFireworks = Cesium.Cartesian3.subtract(this.emitterInitialLocation, cameraOffset, new Cesium.Cartesian3())
        Cesium.Cartesian3.normalize(toFireworks, toFireworks)
        var angle = Cesium.Math.PI_OVER_TWO - Math.acos(Cesium.Cartesian3.dot(toFireworks, Cesium.Cartesian3.UNIT_Z))
        camera.lookUp(angle)
      },
      createFirework(offset, color, bursts) {
        var emitterModelMatrixScratch = new Cesium.Matrix4()
        var position = Cesium.Cartesian3.add(this.emitterInitialLocation, offset, new Cesium.Cartesian3())
        var emitterModelMatrix = Cesium.Matrix4.fromTranslation(position, emitterModelMatrixScratch)
        var particleToWorld = Cesium.Matrix4.multiply(this.modelMatrix, emitterModelMatrix, new Cesium.Matrix4())
        var worldToParticle = Cesium.Matrix4.inverseTransformation(particleToWorld, particleToWorld)

        var size = Cesium.Math.randomBetween(this.minimumExplosionSize, this.maximumExplosionSize)
        var particlePositionScratch = new Cesium.Cartesian3()
        var force = function (particle) {
          var position = Cesium.Matrix4.multiplyByPoint(worldToParticle, particle.position, particlePositionScratch)
          if (Cesium.Cartesian3.magnitudeSquared(position) >= size * size) {
            Cesium.Cartesian3.clone(Cesium.Cartesian3.ZERO, particle.velocity)
          }
        }

        var normalSize = (size - this.minimumExplosionSize) / (this.maximumExplosionSize - this.minimumExplosionSize)
        var minLife = 0.3
        var maxLife = 1.0
        var life = normalSize * (maxLife - minLife) + minLife
        return {
          color,
          image: this.getImage(),
          startColor: color,
          endColor: color.withAlpha(0.0),
          particleLife: life,
          speed: 100.0,
          imageSize: this.particlePixelSize,
          emissionRate: 0,
          emitter: new Cesium.SphereEmitter(0.1),
          bursts: bursts,
          lifetime: this.lifetime,
          updateCallback: force,
          modelMatrix: this.modelMatrix,
          emitterModelMatrix: emitterModelMatrix
        }
      },
      getImage() {
        let particleCanvas = this.particleCanvas
        if (!Cesium.defined(particleCanvas)) {
          particleCanvas = document.createElement('canvas')
          particleCanvas.width = 20
          particleCanvas.height = 20
          var context2D = particleCanvas.getContext('2d')
          context2D.beginPath()
          context2D.arc(8, 8, 8, 0, Cesium.Math.TWO_PI, true)
          context2D.closePath()
          context2D.fillStyle = 'rgb(255, 255, 255)'
          context2D.fill()
          this.particleCanvas = particleCanvas
        }
        return particleCanvas
      },
      onComplete(e) {
        console.log(e)
      },
      onClicked(e) {
        console.log(e)
      },
      unload() {
        this.list.forEach(v => {
          v.unload()
        })
      },
      load() {
        this.list.forEach(v => {
          v.load()
        })
      },
      reload() {
        this.list.forEach(v => {
          v.reload()
        })
      }
    }
  }
</script>
```

:::

### 属性

<!-- prettier-ignore -->
| 属性名 | 类型 | 默认值 | 描述 |
| ------ | ---- | ------ | ---- |
| show | boolean | true | `optional` 是否显示粒子。  |
| updateCallback | Function | | `optional` 更新回调函数。|
| emitter | Cesium.ParticleEmitter |  | `optional` 粒子触发器类型。 |
| modelMatrix | Cesium.Matrix4 | | `optional` 4x4转换矩阵，可将粒子系统从模型转换为世界坐标。 |
| emitterModelMatrix | Cesium.Matrix4 | | `optional` 4x4转换矩阵，用于转换粒子系统局部坐标系内的粒子系统发射器。 |
| emissionRate | number | `5` | `optional` 每秒要发射的粒子数。 |
| bursts | Array | `false` | `optional` ParticleBurst 数组，在周期性时间发射粒子。 |
| loop | boolean | `true` | `optional` 粒子系统完成后是否应循环其爆发。 |
| scale | number | `1.0` | `optional` 设置比例尺，以在其粒子寿命期间应用到粒子图像。 |
| startScale | number |  | `optional` 在粒子寿命开始时应用于粒子图像的初始比例。|
| endScale | number | | `optional` 在粒子寿命结束时应用于粒子图像的最终比例。 |
| color | VcColor\|Array\|string | | `optional` 设置粒子在其粒子寿命期间的颜色。 |
| startColor | VcColor\|Array\|string | | `optional` 粒子在其生命初期的颜色。 |
| endColor| VcColor\|Array\|string | | `optional` 粒子寿命结束时的颜色。|
| image | HTMLImageElement \| HTMLCanvasElement\|string | | `optional` 用于广告牌的URI，HTMLImageElement或HTMLCanvasElement。 |
| imageSize | VcCartesian2 | | `optional` 如果设置，则将覆盖用来缩放粒子图像尺寸（以像素为单位）的minimumImageSize和maximumImageSize输入。 |
| minimumImageSize | VcCartesian2\|Array | | `optional` 设置宽度的最小范围，以高度为单位，在该范围之上可以随机缩放粒子图像的尺寸（以像素为单位）。 |
| maximumImageSize| VcCartesian2\|Array | | `optional` 设置最大边界（宽度乘以高度），在该边界以下可以随机缩放粒子图像的尺寸（以像素为单位）。 |
| speed | number | `1.0` | `optional` 如果设置，则用该值覆盖minimumSpeed和maximumSpeed输入。 |
| minimumSpeed | number | | `optional` 设置以米/秒为单位的最小范围，在该范围上可以随机选择粒子的实际速度。|
| maximumSpeed | number | | `optional` 设置以米/秒为单位的最大范围，在该范围内将随机选择粒子的实际速度。 |
| lifetime | number | | `optional` 粒子系统发射粒子的时间（以秒为单位）。 |
| particleLife | number | `5.0` | `optional` 如果设置，则使用此值覆盖minimumParticleLife和maximumParticleLife输入。 |
| minimumParticleLife | number | | `optional` 设置以秒为单位的粒子寿命的可能持续时间的最小范围，在该范围内可以随机选择粒子的实际寿命。 |
| maximumParticleLife | number | | `optional` 设置以秒为单位的粒子生命的可能持续时间的最大范围，在该范围内将随机选择粒子的实际生命。 |
| mass | number | `1.0` | `optional` 设置粒子的最小和最大质量（以千克为单位）。 |
| minimumMass | number | | `optional` 设置粒子质量的最小范围（以千克为单位）。 粒子的实际质量将被选择为高于此值的随机量。 |
| maximumMass | number | | `optional` 设置最大粒子质量（以千克为单位）。 粒子的实际质量将选择为低于此值的随机量。 |
| enableMouseEvent | boolean | `true` | `optional` 指定鼠标事件是否生效。 |

### 事件

| 事件名       | 参数                                    | 描述                       |
| ------------ | --------------------------------------- | -------------------------- |
| beforeLoad   | (instance: VcComponentInternalInstance) | 对象加载前触发。           |
| ready        | (readyObj: VcReadyObject)               | 对象加载成功时触发。       |
| destroyed    | (instance: VcComponentInternalInstance) | 对象销毁时触发。           |
| readyPromise |                                         | 模型对象可用时触发。       |
| mousedown    | (evt: VcPickEvent)                      | 鼠标在该图元上按下时触发。 |
| mouseup      | (evt: VcPickEvent)                      | 鼠标在该图元上弹起时触发。 |
| click        | (evt: VcPickEvent)                      | 鼠标单击该图元时触发。     |
| clickout     | (evt: VcPickEvent)                      | 鼠标单击该图元外部时触发。 |
| dblclick     | (evt: VcPickEvent)                      | 鼠标左键双击该图元时触发。 |
| mousemove    | (evt: VcPickEvent)                      | 鼠标在该图元上移动时触发。 |
| mouseover    | (evt: VcPickEvent)                      | 鼠标移动到该图元时触发。   |
| mouseout     | (evt: VcPickEvent)                      | 鼠标移出该图元时触发。     |

### 参考

- 官方文档： **[ParticleSystem](https://cesium.com/docs/cesiumjs-ref-doc/ParticleSystem.html)**
