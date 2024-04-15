<template>
	<div class="container">
		<h1 style="text-align: center">一个基于Farbic.js的在线图片标注组件</h1>
		<Row :gutter="20">
			<Col :span="20">
				<Annotation ref="myTagging" style="width: 100%" img-src="http://www.news.cn/photo/20240410/747c2e6e95ec4b9f9d9e437bbe91ab2a/20240410747c2e6e95ec4b9f9d9e437bbe91ab2a_202404103ec0b250e00a47b09dcff8e010a5c6f5.jpg" />
			</Col>
			<Col :span="4">
				<div style="text-align: center">
					<RadioGroup @change="tpChange" v-model:value="currentTagTp" style="text-align: left">
						<Radio :style="radioStyle" value="rectangle">矩形标注</Radio>
						<Radio :style="radioStyle" value="polygn">多边形标注</Radio>
					</RadioGroup>
					<p style="margin-top: 50px">
						<a-button type="primary" @click="chooseLabel({ id: 1, name: 'test标注' })"> 使用test标注 </a-button>
					</p>
				</div>
			</Col>
		</Row>
	</div>
</template>

<script setup lang="ts">
import { reactive, ref } from 'vue'
import Annotation from './components/annotation.vue'
import { Row, Col, RadioGroup, Radio } from 'ant-design-vue'

const radioStyle = reactive({
	display: 'block',
	height: '40px',
	lineHeight: '40px'
})

const currentTagTp = ref('rectangle')

const myTagging = ref()

const tpChange = e => {
	if (e && e.target.value) {
		myTagging.value.changeDrawType(e.target.value)
	}
}

export interface annoItem {
	id: string | number
	name: string
}

const currentAnno = reactive({
	data: <annoItem>{}
})

const chooseLabel = (item: annoItem) => {
	if (item.id === currentAnno.data.id) return
	currentAnno.data = item
	myTagging.value.setCurrentTagging(item)
}
</script>

<style scoped>
.container {
	margin: 0 auto;
	width: 80%;
}
</style>
