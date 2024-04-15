<template>
	<div class="container">
		<h1 style="text-align: center">一个基于Farbic.js的在线图片标注组件</h1>
		<Row :gutter="20">
			<Col :span="20">
				<Annotation ref="myTagging" style="width: 100%" img-src="https://zos.alipayobjects.com/rmsportal/jkjgkEfvpUPVyRjUImniVslZfWPnJuuZ.png?x-oss-process=image/blur,r_50,s_50/quality,q_1/resize,m_mfit,h_200,w_200" />
			</Col>
			<Col :span="4">
				<div style="text-align: center">
					<RadioGroup @change="tpChange" v-model:value="currentTagTp" style="text-align: left">
						<Radio :style="radioStyle" value="rectangle">矩形标注</Radio>
						<Radio :style="radioStyle" value="polygn">多边形标注</Radio>
					</RadioGroup>
					<p style="margin-top: 50px">
						<a-button type="primary" @click="chooseLabel({ id: 1, name: 'test标注' })"> test标注 </a-button>
					</p>
					<p style="text-align: left; word-break: break-all">
						<span>提示：<br /><br />点击[test标注]按钮再点击图片测试标注。<br /><br />原业务需求标签和标注唯一，所以在只有'test标注'标签的情况下只能有一个标注。测试标注时如果已有标注，删除了重新标注就行。</span>
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
