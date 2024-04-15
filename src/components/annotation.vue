<template>
	<div style="width: 100%">
		<canvas id="myCanvas"></canvas>
	</div>
</template>

<script lang="ts" setup>
import { annoItem } from '../App.vue'
import { fabric } from 'fabric'
import { reactive, onMounted, ref } from 'vue'

const props = defineProps<{
	imgSrc: string // 待标注背景图片
	data?: Array<any> // 需要预渲染的标注数据
}>()

let myCanvas = ref()

const init = () => {
	// 整体初始化
	initFabricControl()
}

onMounted(() => {
	if (!props.imgSrc) return
	init()
})

const emit = defineEmits([
	'changeLabel', // 标注标签更改事件
	'taggingChange', // 单体标注更改事件
	'hasChange' // 整体标注数据是否有改变 用来判断是否向用户弹出保存当前修改的弹窗
])

// 当前图片的标注数据是否有改变
// 用来判断是否向用户弹出保存修改的弹窗

// 是否要记录标注数据改变
let toRecord = false

// 提醒父组件数据改变
const hasChange = () => {
	emit('hasChange')
}

// 初始化容器宽高,初始化当前背景图片相对于原始图片的缩放比例
let containerWidth = 0
let containerHeight = 0
let adapterRatio = 0

const initFabricControl = () => {
	let canvas = document.querySelector('#myCanvas')

	fabric.Object.prototype.perPixelTargetFind = true
	fabric.Object.prototype.targetFindTolerance = 0
	fabric.Object.prototype.hasRotatingPoint = false
	// 初始化缩放控件的属性
	fabric.Object.prototype.cornerColor = 'white'
	fabric.Object.prototype.cornerStyle = 'circle'
	fabric.Object.prototype.transparentCorners = false
	// 删除按钮 没研究 使用原demo
	fabric.Object.prototype.controls.deleteControl = new fabric.Control({
		x: 0.5,
		y: -0.5,
		offsetY: 16,
		cursorStyle: 'pointer',
		mouseUpHandler: deleteObject,
		render: renderIcon,
		cornerSize: 14
	})

	// 控制背景不缩放不选中
	myCanvas.value = new fabric.Canvas(canvas, {
		backgroundVpt: false,
		selection: false
	})

	// 加载待标注图片作为背景图片 根据容器宽度适应高度
	fabric.Image.fromURL(props.imgSrc, img => {
		if (!img || !img.width || !img.height) return

		// 获得容器宽度 -> 背景图片宽度
		containerWidth = canvas?.parentNode.offsetWidth

		//  背景图片宽度 / 原始图片宽度 = 缩放比例
		adapterRatio = containerWidth / img.width

		// 原始图片宽高比
		let ratio = img.width / img.height

		// 根据比例计算出背景图片保持原图片比例需要的容器高度
		containerHeight = containerWidth / ratio

		myCanvas.value.setWidth(containerWidth)
		myCanvas.value.setHeight(containerHeight)

		myCanvas.value.setBackgroundImage(img, myCanvas.value.renderAll.bind(myCanvas.value), {
			scaleX: containerWidth / img.width, //计算出图片要拉伸的宽度
			scaleY: containerHeight / img.height //计算出图片要拉伸的高度
		})

		// 渲染已标注数据
		drawPreTagging()
	})

	// 监听Mouse按下事件
	myCanvas.value.on('mouse:down', mousedown)

	// 监听Mouse移动事件
	myCanvas.value.on('mouse:move', mousemove)

	// 重置
	reset()
}

// 当前标注Id
const currentTaggingId = ref<string | number>('')

// 当前标注标签
const currentTaggingName = ref('')

const reset = () => {
	currentTaggingId.value = ''
	clearAll()
	state.polygonMode = false
	state.drawType = 'rectangle'
}

// 预加载已有的标注数据
const drawPreTagging = () => {
	if (!props.data || !props.data.length) {
		toRecord = true
		return
	}

	let preList = JSON.parse(JSON.stringify(props.data))

	preList.map(p => {
		currentTaggingId.value = p.id
		currentTaggingName.value = p.name

		if (p.type === 1) {
			p.sx1 *= adapterRatio
			p.sy1 *= adapterRatio
			p.sx2 *= adapterRatio
			p.sy2 *= adapterRatio

			state.mouseFrom.x = p.sx1
			state.mouseFrom.y = p.sy1
			state.mouseTo.x = p.sx2
			state.mouseTo.y = p.sy2

			generateRect()
		} else if (p.type === 2) {
			try {
				let temp = JSON.parse(p.pointJson)
				temp = temp.map(t => {
					t.left = adapterRatio * t.x
					t.top = adapterRatio * t.y
					return t
				})

				state.pointArray = temp

				generatePolygon()
			} catch {}
		}
	})

	// 不对任何Canvas对象聚焦
	myCanvas.value.discardActiveObject()
	reset()

	// 开启记录
	toRecord = true
}

const state = reactive({
	isScaling: false,
	imgPoint: { x: 0, y: 0 },
	realPoint: { x: 0, y: 0 },
	mouseFrom: { x: 0, y: 0 } as canvasPoint,
	mouseTo: { x: 0, y: 0 } as canvasPoint,
	rectPath: '' as string, //矩形绘制路径
	color: '#0960bd', //画笔颜色
	drawWidth: 2, //笔触宽度
	drawingObject: null as any, //当前绘制对象
	//moveCount: 1, //绘制移动计数器

	drawType: 'rectangle',
	pointArray: [] as any,
	line: {} as canvasPoint,
	lineArray: [] as canvasPoint[],
	activeLine: '' as any,
	activeShape: false as any,
	//polygon 相关参数
	polygonMode: false as boolean,
	doDrawing: false as boolean, // 绘制状态,
	currentStep: 'pre'
})

// 更改当前标注类型
const changeDrawType = (tp: string) => {
	if (tp === 'polygn') {
		drawPolygon()
	} else {
		clearAll()
		state.polygonMode = false
		state.drawType = 'rectangle'
	}
}

// 设置当前标注
const setCurrentTagging = (item: annoItem) => {
	if (currentTaggingId.value === item.id) return

	currentTaggingId.value = item.id
	currentTaggingName.value = item.name
	let tagging = rectList.get(item.id)
	if (!tagging) return

	myCanvas.value.setActiveObject(tagging)
	myCanvas.value.requestRenderAll()
}

const drawPolygon = () => {
	state.drawType = 'polygn'
	state.polygonMode = true
	//这里画的多边形，由顶点与线组成
	state.pointArray = new Array() // 顶点集合
	state.lineArray = new Array() //线集合
}

// 重置并清除无关要素 主要清除鼠标移动时实时生成的标注框
const clearAll = () => {
	// 标注元素已添加objectId 以此删除无关要素
	let objects = myCanvas.value.getObjects()
	if (objects && objects.length) {
		// 开发时后端接口有id为0情况  此处无关紧要
		objects.map(x => {
			if (!x.objectId && x.objectId !== 0) myCanvas.value.remove(x)
		})
	}

	// 回到矩形标注的准备状态
	state.currentStep = 'pre'
	// 设置标注状态为否
	state.doDrawing = false
	// 回到点位标注的准备状态
	resetPolygon()
}

const resetPolygon = () => {
	state.activeLine = null
	state.activeShape = null
	state.pointArray = []
	state.lineArray = []
}

// 借鉴前人 不知此处的意义
const transformMouse = (mouseX, mouseY) => {
	return { x: mouseX / 1, y: mouseY / 1 }
}

// 用map记录一下添加的标注元素 以备处理
const rectList = new Map()

// 判断鼠标是否落在已有选区 是否点位落在了已有标注数据上
const isRepeatClick = e => {
	if (!rectList || !rectList.size) return false

	let isRepeat = false

	for (let [_key, value] of rectList) {
		// 判断点位是否落在选区上
		let point = new fabric.Point(e.e.offsetX, e.e.offsetY)

		let result = value.containsPoint(point)

		// 如果落在选区 告诉父组件定位原标注 重新设置当前激活的标注ID标签等属性
		if (result) {
			myCanvas.value.setActiveObject(value)

			emit('changeLabel', value)
			break
		}
	}

	return isRepeat
}

// 鼠标按下
const mousedown = e => {
	isRepeatClick(e)

	// 如果当前没有选中标注 (实际业务需求参看Readme 使用时请调整) 返回
	if (!currentTaggingId.value && currentTaggingId.value !== 0) return
	// 如果已经有当前标注数据 返回
	if (rectList.has(currentTaggingId.value)) return

	// 点位标注
	if (state.drawType !== 'rectangle') {
		myCanvas.value.skipTargetFind = false
		try {
			let len = 4

			if (state.polygonMode) {
				if (state.pointArray.length === 0) {
					let isRepeat = isRepeatClick(e)
					if (isRepeat) return
				}
				addPoint(e)
			}

			if (state.pointArray.length === len && state.polygonMode) {
				generatePolygon()
			}
		} catch (error) {
			console.log(error)
		}
	}
	// 矩形标注
	else {
		// 有鼠标移动时显示的矩形框 先删除
		if (state.drawingObject) {
			myCanvas.value.remove(state.drawingObject)
		}

		// 准备阶段
		if (state.currentStep === 'pre') {
			let isRepeat = isRepeatClick(e)
			if (isRepeat) return
			var xy = e.pointer || transformMouse(e.e.offsetX, e.e.offsetY)
			state.mouseFrom.x = xy.x
			state.mouseFrom.y = xy.y

			state.doDrawing = true

			state.currentStep = 'ing'
		}
		// 已点击 进行中阶段
		else if (state.currentStep === 'ing') {
			let xy = e.pointer || transformMouse(e.e.offsetX, e.e.offsetY)
			state.mouseTo.x = xy.x
			state.mouseTo.y = xy.y

			generateRect()
		}
	}
}

const generateRect = () => {
	let left = state.mouseTo.x < state.mouseFrom.x ? state.mouseTo.x : state.mouseFrom.x,
		top = state.mouseTo.y < state.mouseFrom.y ? state.mouseTo.y : state.mouseFrom.y,
		width = Math.abs(state.mouseTo.x - state.mouseFrom.x),
		height = Math.abs(state.mouseTo.y - state.mouseFrom.y)

	if (width + left > containerWidth) {
		width = containerWidth - left
	}

	if (height + top > containerHeight) {
		height = containerHeight - top
	}

	state.doDrawing = false

	let rect = ref()

	rect.value = new fabric.Rect({
		objectId: currentTaggingId.value,
		left,
		top,
		// fill: currentTaggingId.value === 0 ? 'rgba(237, 111, 111, 0.8)' : 'rgba(9, 96, 189, 0.8)',
		fill: 'rgba(255, 255, 255, 0.01)',
		width,
		height,
		objectCaching: false,
		stroke: currentTaggingId.value === 0 ? '#ED6f6f' : state.color,
		strokeWidth: state.drawWidth,
		lockRotation: true,
		zIndex: 2
	})

	var text = new fabric.Textbox(currentTaggingName.value, {
		tp: 'text',
		objectId: currentTaggingId.value,
		fontSize: 20,
		left,
		top: top + height / 2 - 7,
		fill: '#0960bd',
		fontWeight: 'bold',
		textAlign: 'center',
		width,
		height,
		splitByGrapheme: true,
		selectable: false,
		event: false,
		zIndex: -1
	})

	rect.value.on('moving', options => {
		fixUpMoveOverflow()
		clearAll()

		if (toRecord) hasChange()
		return false
	})

	rect.value.on('scaling', options => {
		fixUpMoveOverflow()
		clearAll()

		if (toRecord) hasChange()
		return false
	})

	rect.value.on('modified', options => {
		objectChange()
		if (toRecord) hasChange()
	})

	rect.value.on('added', options => {
		objectChange()
		if (toRecord) hasChange()
	})

	rect.value.setControlVisible('mtr', false)

	myCanvas.value.add(rect.value)
	myCanvas.value.add(text)
	myCanvas.value.setActiveObject(rect.value)

	rectList.set(currentTaggingId.value, rect.value)

	emit('taggingChange', { tp: 'add', id: currentTaggingId.value })

	clearAll()
}

const addPoint = e => {
	let random = Math.floor(Math.random() * 10000)
	let id = new Date().getTime() + random
	let circle: any = new fabric.Circle({
		radius: 5,
		fill: '#ffffff',
		stroke: '#333333',
		strokeWidth: 0.5,
		left: (e.pointer.x || e.e.layerX) / myCanvas.value.getZoom(),
		top: (e.pointer.y || e.e.layerY) / myCanvas.value.getZoom(),
		selectable: false,
		hasBorders: false,
		hasControls: false,
		originX: 'center',
		originY: 'center',
		id: id,
		objectCaching: false
	})
	if (state.pointArray.length == 0) {
		circle.set({
			fill: '#00FFFF'
		})
	}

	var points = [(e.pointer.x || e.e.layerX) / myCanvas.value.getZoom(), (e.pointer.y || e.e.layerY) / myCanvas.value.getZoom(), (e.pointer.x || e.e.layerX) / myCanvas.value.getZoom(), (e.pointer.y || e.e.layerY) / myCanvas.value.getZoom()]

	state.line = new fabric.Line(points, {
		strokeWidth: 2,
		fill: '#999999',
		stroke: '#999999',
		class: 'line',
		originX: 'center',
		originY: 'center',
		selectable: false,
		hasBorders: false,
		hasControls: false,
		evented: false,

		objectCaching: false
	})
	if (state.activeShape) {
		let pos = myCanvas.value.getPointer(e.e)
		let points = state.activeShape.get('points')
		points.push({
			x: pos.x,
			y: pos.y
		})
		var polygon = new fabric.Polygon(points, {
			stroke: '#333333',
			strokeWidth: 1,
			fill: '#cccccc',
			opacity: 0.3,
			selectable: false,
			hasBorders: false,
			hasControls: false,
			evented: false,
			objectCaching: false
		})
		myCanvas.value.remove(state.activeShape)
		myCanvas.value.add(polygon)
		state.activeShape = polygon
		myCanvas.value.renderAll()
	} else {
		var polyPoint = [
			{
				x: (e.pointer.x || e.e.layerX) / myCanvas.value.getZoom(),
				y: (e.pointer.y || e.e.layerY) / myCanvas.value.getZoom()
			}
		]
		var polygon = new fabric.Polygon(polyPoint, {
			stroke: '#333333',
			strokeWidth: 1,
			fill: '#cccccc',
			opacity: 0.3,
			selectable: false,
			hasBorders: false,
			hasControls: false,
			evented: false,
			objectCaching: false
		})
		state.activeShape = polygon
		myCanvas.value.add(polygon)
	}
	state.activeLine = state.line

	state.pointArray.push(circle)
	state.lineArray.push(state.line)
	myCanvas.value.add(state.line)
	myCanvas.value.add(circle)
}

const clearPolygonLines = () => {
	let points: any = new Array()
	state.pointArray.forEach((point: any) => {
		points.push({
			x: point.left,
			y: point.top
		})
		myCanvas.value.remove(point)
	})
	state.lineArray.forEach(line => {
		myCanvas.value.remove(line)
	})
	myCanvas.value.remove(state.activeShape).remove(state.activeLine)
	return points
}

// 绘制不规则四边形
const generatePolygon = () => {
	let points = clearPolygonLines()
	let polygon = ref()

	polygon.value = new fabric.Polygon(sortPoints(points), {
		objectId: currentTaggingId.value,
		stroke: currentTaggingId.value === 0 ? '#ED6f6f' : state.color,
		strokeWidth: state.drawWidth,
		// fill: currentTaggingId.value === 0 ? 'rgba(237, 111, 111, 0.8)' : 'rgba(9, 96, 189, 0.8)',
		fill: 'rgba(255, 255, 255, 0.01)',
		opacity: 1,
		objectCaching: false,
		lockRotation: true,
		tp: 'polygn'
	})

	console.log(polygon.value)

	var text = new fabric.Textbox(currentTaggingName.value || '', {
		tp: 'text',
		objectId: currentTaggingId.value,
		fontSize: 20,
		left: polygon.value.left, // 文字的初始x坐标
		top: polygon.value.top + polygon.value.height / 2 - 7, // 文字的初始y坐标
		//padding: 20, // 文字与边框的距离
		fill: '#0960bd',
		fontWeight: 'bold',
		textAlign: 'center',

		width: polygon.value.width,
		height: polygon.value.height,
		splitByGrapheme: true,
		selectable: false,
		event: false,
		zIndex: -1
	})

	polygon.value.on('moving', options => {
		fixUpMoveOverflow()
		clearAll()

		if (toRecord) hasChange()
		return false
	})

	polygon.value.on('scaling', options => {
		fixUpMoveOverflow()
		clearAll()

		if (toRecord) hasChange()

		return false
	})

	polygon.value.on('modified', options => {
		objectChange()

		if (toRecord) hasChange()
	})

	polygon.value.on('added', options => {
		objectChange()

		if (toRecord) hasChange()
	})

	polygon.value.setControlVisible('mtr', false)

	myCanvas.value.add(polygon.value)
	myCanvas.value.add(text)

	rectList.set(currentTaggingId.value, polygon.value)

	myCanvas.value.setActiveObject(polygon.value)

	emit('taggingChange', { tp: 'add', id: currentTaggingId.value })

	clearAll()

	drawPolygon()
}

const fixUpMoveOverflow = () => {
	let rect = myCanvas.value.getActiveObject()

	let left = rect.left,
		top = rect.top,
		width = rect.width * rect.scaleX,
		height = rect.height * rect.scaleY

	if (left < 0) {
		rect.set({ left: 0 })
		left = 0
	}

	if (top < 0) {
		rect.set({ top: 0 })
		top = 0
	}

	if (width + left > containerWidth) {
		if (width > containerWidth) {
			width = containerWidth
		}

		rect.set({ left: containerWidth - width, scaleX: 1, width })
	}

	if (height + top > containerHeight) {
		if (height > containerHeight) {
			height = containerHeight
		}
		// 如果矩形顶部触及边界，将其设置在边界上
		rect.set({ top: containerHeight - height, scaleY: 1, height })
	}
}

const mousemove = e => {
	if (!state.doDrawing) {
		//减少绘制频率
		return
	}
	//state.moveCount++
	var xy = e.pointer || transformMouse(e.e.offsetX, e.e.offsetY)
	if (true || (xy.y >= 0 && xy.x <= containerWidth && xy.y >= 0 && xy.y <= containerHeight)) {
		let realX = 0
		let realY = 0

		if (xy.x >= containerWidth) {
			realX = containerWidth
		} else {
			if (xy.x < 0) {
				realX = 0
			}
			realX = xy.x
		}

		if (xy.y >= containerHeight) {
			realY = containerHeight
		} else {
			if (xy.y < 0) {
				realY = 0
			}
			realY = xy.y
		}

		//console.log(containerWidth, xy.x, 'x')
		state.mouseTo.x = realX
		state.mouseTo.y = realY
		// 矩形
		if (state.drawType == 'rectangle') {
			if (state.currentStep === 'ing') {
				drawing(e)
			} else {
				// clearAll();
			}
		} else if (state.drawType !== 'rectangle') {
			if (state.activeLine && state.activeLine.class == 'line') {
				var pointer = myCanvas.value.getPointer(e.e)
				state.activeLine.set({ x2: pointer.x, y2: pointer.y })

				var points = state.activeShape.get('points')
				points[state.pointArray.length] = {
					x: pointer.x,
					y: pointer.y,
					zIndex: 1
				}
				state.activeShape.set({
					points: points
				})
				myCanvas.value.renderAll()
			}
			myCanvas.value.renderAll()
		}
	} else {
		// clearAll();
	}
}

const drawing = _event => {
	//if (state.isScaling) return
	if (state.drawingObject) {
		myCanvas.value.remove(state.drawingObject)
	}
	let canvasObject = null

	let mouseFrom: any = {},
		mouseTo: any = {}

	//console.log(state.mouseFrom, state.mouseTo)

	if (state.mouseFrom.x > state.mouseTo.x) {
		//let temp = JSON.parse(JSON.stringify(state.mouseFrom))
		mouseFrom = state.mouseTo
		mouseTo = state.mouseFrom
	} else {
		mouseFrom = state.mouseFrom
		mouseTo = state.mouseTo
	}

	let left = state.mouseTo.x < state.mouseFrom.x ? state.mouseTo.x : state.mouseFrom.x,
		top = state.mouseTo.y < state.mouseFrom.y ? state.mouseTo.y : state.mouseFrom.y

	var path = 'M ' + mouseFrom.x + ' ' + mouseFrom.y + ' L ' + mouseTo.x + ' ' + mouseFrom.y + ' L ' + mouseTo.x + ' ' + mouseTo.y + ' L ' + mouseFrom.x + ' ' + mouseTo.y + ' L ' + mouseFrom.x + ' ' + mouseFrom.y + ' z'

	state.rectPath = path
	canvasObject = new fabric.Path(path, {
		left: left,
		top: top,
		stroke: state.color,
		selectable: false,
		strokeWidth: state.drawWidth,
		//fill: 'rgba(9, 96, 189, 0.5)',
		fill: 'transparent',
		hasControls: false
	})

	if (canvasObject) {
		myCanvas.value.add(canvasObject)
		state.drawingObject = canvasObject
	}
}

var img = document.createElement('img')
img.src = "data:image/svg+xml,%3C%3Fxml version='1.0' encoding='utf-8'%3F%3E%3C!DOCTYPE svg PUBLIC '-//W3C//DTD SVG 1.1//EN' 'http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd'%3E%3Csvg version='1.1' id='Ebene_1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' x='0px' y='0px' width='595.275px' height='595.275px' viewBox='200 215 230 470' xml:space='preserve'%3E%3Ccircle style='fill:%23F44336;' cx='299.76' cy='439.067' r='218.516'/%3E%3Cg%3E%3Crect x='267.162' y='307.978' transform='matrix(0.7071 -0.7071 0.7071 0.7071 -222.6202 340.6915)' style='fill:white;' width='65.545' height='262.18'/%3E%3Crect x='266.988' y='308.153' transform='matrix(0.7071 0.7071 -0.7071 0.7071 398.3889 -83.3116)' style='fill:white;' width='65.544' height='262.179'/%3E%3C/g%3E%3C/svg%3E"

const renderIcon = (ctx, left, top, styleOverride, fabricObject) => {
	var size = 24
	ctx.save()
	ctx.translate(left, top)
	ctx.rotate(fabric.util.degreesToRadians(fabricObject.angle))
	ctx.drawImage(img, -size / 2, -size / 2, size, size)
	ctx.restore()
}

// 删除Object
const deleteObject = (eventData, transform) => {
	let target = myCanvas.value.getActiveObject()
	myCanvas.value.remove(target)

	let all = myCanvas.value.getObjects()

	let textBox = all.find(x => x.objectId == target.objectId)
	if (textBox) {
		myCanvas.value.remove(textBox)
	}

	myCanvas.value.requestRenderAll()

	emit('taggingChange', { tp: 'del', id: target.objectId })

	clearAll()

	rectList.delete(target.objectId)

	hasChange()
}

// 处理标注标签的文字自动跟随对象的移动或缩放
const objectChange = () => {
	let obj = myCanvas.value.getActiveObject()
	if (!obj) return
	let all = myCanvas.value.getObjects()

	let textBox = all.find(x => x.objectId == obj.objectId && x.tp === 'text')

	if (textBox) {
		textBox.set({
			left: obj.left,
			top: obj.top + (obj.height * obj.scaleY) / 2 - 7,
			width: obj.width * obj.scaleX,
			heigth: obj.height * obj.scaleY
		})
	}
}

// 坐标点排序
const sortPoints = (aCoord: any) => {
	let points = JSON.parse(JSON.stringify(aCoord))
	console.log(points)
	// (tl tr) (bl,br)
	const poiT: canvasPoint = linesIntersection(points[0] || points.tl, points[1] || points.tr, points[2] || points.br, points[3] || points.bl)
	// (tl br) (tr,bl)
	const poiB: canvasPoint = linesIntersection(points[0] || points.tl, points[3] || points.bl, points[2] || points.br, points[1] || points.tr)
	if (poiT?.x != -1) {
		// 第二个和第三个坐标互换
		changePoint(points[1] || points.tr, points[2] || points.br)
	} else if (poiB?.x != -1) {
		// 第一个和第四个坐标互换
		changePoint(points[0] || points.tl, points[3] || points.bl)
		// 第二个和第三个坐标互换
		changePoint(points[1] || points.tr, points[2] || points.br)
		// 第一个和第二个坐标互换
		changePoint(points[0] || points.tl, points[1] || points.tr)
	}
	return points
}

const changePoint = (poi1, poi2) => {
	let temp = { x: poi1.x, y: poi1.y }
	poi1.x = poi2.x
	poi1.y = poi2.y

	poi2.x = temp.x
	poi2.y = temp.y
}

function linesIntersection(tl, tr, br, bl) {
	let tn1 = tr.y - tl.y,
		ty1 = tl.x - tr.x
	let tn2 = br.y - bl.y,
		ty2 = bl.x - br.x
	let denominator = tn1 * ty2 - ty1 * tn2
	if (denominator == 0) {
		return { x: -1, y: -1 }
	}
	let distC_N2 = tn2 * bl.x + ty2 * bl.y
	let distA_N2 = tn2 * tl.x + ty2 * tl.y - distC_N2
	let distB_N2 = tn2 * tr.x + ty2 * tr.y - distC_N2

	if (distA_N2 * distB_N2 >= 0) {
		return { x: -1, y: -1 }
	}
	let distA_N1 = tn1 * tl.x + ty1 * tl.y
	let distC_N1 = tn1 * bl.x + ty1 * bl.y - distA_N1
	let distD_N1 = tn1 * br.x + ty1 * br.y - distA_N1
	if (distC_N1 * distD_N1 >= 0) {
		return { x: -1, y: -1 }
	}
	//计算交点坐标
	let fraction = distA_N2 / denominator
	let dx = fraction * ty1,
		dy = -fraction * tn1
	return { x: tl.x + dx, y: tl.y + dy }
}

// 调用此方法可得到最终所有标注结果点位
const getAllObjects = () => {
	// 再更新一次画布 忘了有没有意义 管他呢
	myCanvas.value.requestRenderAll()

	// 获得所有Object
	let all = myCanvas.value.getObjects() || []

	if (all.length) {
		// 仅获得标注 排除无关元素和仅用来展示的文字
		all = all.filter(x => (x.objectId || x.objectId === 0) && x.tp !== 'text')

		all = all.map(x => {
			let obj: any = {
				id: x.objectId
			}

			// farbic.js 不支持获取移动或者缩放后的点位标注的坐标，做了处理
			if (x.tp === 'polygn') {
				obj.type = 2

				let minXIndex = 0
				let minYIndex = 0
				let tempMinX = 0
				let tempMinY = 0
				let tempIndex = -1

				let points = x.points.map((item: any) => {
					tempIndex++
					if (!tempMinX) {
						tempMinX = item.x
					}
					if (!tempMinY) {
						tempMinY = item.y
					}

					if (item.x < tempMinX) {
						tempMinX = item.x
						minXIndex = tempIndex
					}

					if (item.y < tempMinY) {
						tempMinY = item.y
						minYIndex = tempIndex
					}

					item.x *= x.scaleX
					item.y *= x.scaleY
					return item
				})

				let offsetX = points[minXIndex].x - x.left
				let offsetY = points[minYIndex].y - x.top

				for (let i = 0; i < points.length; i++) {
					if (i === minXIndex) {
						points[i].x = x.left
					} else {
						points[i].x = points[i].x - offsetX
					}
					if (i === minYIndex) {
						points[i].y = x.top
					} else {
						points[i].y = points[i].y - offsetY
					}
				}

				points = x.points.map(item => {
					item.x /= adapterRatio
					item.y /= adapterRatio

					item.x = item.x.toFixed(8)
					item.y = item.y.toFixed(8)
					return item
				})

				obj.pointJson = JSON.stringify(points)
			} else {
				obj.width = ((x.width / adapterRatio) * x.scaleX).toFixed(8)
				obj.height = ((x.height / adapterRatio) * x.scaleY).toFixed(8)
				obj.sx1 = (x.lineCoords.tl.x / adapterRatio).toFixed(8)
				obj.sy1 = (x.lineCoords.tl.y / adapterRatio).toFixed(8)
				obj.sx2 = (x.lineCoords.br.x / adapterRatio).toFixed(8)
				obj.sy2 = (x.lineCoords.br.y / adapterRatio).toFixed(8)
				obj.type = 1
			}

			return obj
		})
	}
	return all
}

defineExpose({
	changeDrawType, // 更改标注方式
	setCurrentTagging, // 设置当前标注的属性
	getAllObjects // 获取最终的标注结果
})
</script>

<style scoped>
:deep(.canvas-container) {
	width: 100% !important;
}
</style>
