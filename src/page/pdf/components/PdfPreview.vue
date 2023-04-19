

<template>
	<!-- 布局控件 -->
	<el-row type="flex" v-loading="loading">
		<!-- 左边预览pdf位置宽一些 -->
		<el-col :span="24">
			<!-- PDF 预览 组件最外层 -->
			<div class="pdfCanvasWrappers">
				<!-- PDF内部滚动  滚动核心 -->
				<div id="pdfCanvasInner" class="pdfCanvasInner">


					<!-- pdf 滚动 定位使用的是sticky 现代浏览器好用 -->
					<div class="pdfPreview_toolsBar">
						<div>
							<!-- 上一页 -->
							<i class="buttonItem el-icon-back  buttonIcon" :disabled="pageNum <= 1" @click="onPrevPage">
							</i>

							<!-- 跳页 -->
							<span class="buttonItem">
								<!-- 重写样式 -->
								<el-input class="pdfPreview_toolsBar-pageNum" onkeyup="value=value.replace(/[^\d]/g,0)"
									@change="pageChange" v-model="pageNum"></el-input>
								<span style="display:inline-block;  position: relative; top: -1px; font-size: 14px;"> /
								</span>
								<span style="display:inline-block; padding:0px 8px; font-size: 14px;"> {{ numPages }}
								</span>
							</span>

							<!-- 下一页 -->
							<i class="buttonItem el-icon-right  buttonIcon" :disabled="pageNum >= numPages"
								@click="onNextPage">
							</i>

						</div>
					</div>


					<!-- pdf Canvas   画布,pdfCanvas画布 要和pdf保持一致, 这样position的定位才能准确 -->
					<div class="pdfCanvas">
						<!-- canvas画布遍历  一页pdf渲染一页 -->
						<div v-for="n in numPages" :key="n" :style="{ height: pageHeight + 'px' }">
							<canvas :id="'pdf-canvas-' + n"></canvas>
						</div>
					</div>


				</div>
				<!-- PDF内部滚动  滚动核心 结束 -->
			</div>
			<!--PDF 预览 组件最外层 结束  -->
		</el-col>

	</el-row>
</template>

<script>
// pdf.js 在vue兼容用法
import * as pdfjsLib from "pdfjs-dist/legacy/build/pdf.js";
window.pdfjsWorker = require("pdfjs-dist/build/pdf.worker");


export default {
	data() {
		return {
			loading: false,
			pdfDoc: null,   //获取到的pdf对象，  这里面是pdf.js返回的对象， 看接口看的各种方法就在里面
			pageHeight: 0,  //PDF 每页高度， 这个很关键， 各种定位都和这个高度有关系
			numPages: 0,	//pdfjs返回的pdf总页数
			pageNum: 1,		//pdf页码
			previewLink: ''
		};
	},
	props: {
		//pdf的url, 由父组件传递，后面可能通过get进行参数传递
		url: {
			type: String,
			required: true,
		},
		// 缩放比例, 由父组件传递，后面可能通过get进行参数传递
		scale: {
			type: Number,
			default: 1.0,
		},
	},
	async mounted() {
		const previewLinkId = localStorage.getItem('previewLink')
		this.previewLink = previewLinkId ? true : false
		//加载完结构之后， 加载pdf
		let url;
		this.loading = true

		await this.loadPdf(url)

		// 遍历PDF的页码 和 视图的canvas 通过 ID 一一对应。
		for (let i = 1; i <= this.numPages; i++) {
			this.renderPage(i, `pdf-canvas-${i}`);
		}
		this.loading = false

		//监听元素滚动缩放
		this.pdfCanvasInner = document.getElementById("pdfCanvasInner");
		this.pdfCanvasInner.addEventListener("scroll", this.onScroll);


	},


	//  这里的watch没有参与逻辑， 只是为了看属性变化
	watch: {
		//监听页码变化
		pageNum: {
			immediate: true,
			deep: true,
			handler(val) {
				// 当前时间戳
				console.log("监听页码", val);
			},
		},
	},
	methods: {

		// 加载pdf
		async loadPdf(previewLink) {
			const loadingTask = pdfjsLib.getDocument(previewLink ? previewLink : this.url);
			this.pdfDoc = await loadingTask.promise;
			this.numPages = this.pdfDoc.numPages;
			this.pageHeight = await this.getPageHeight(1);
		},

		// 获取pdf页面高度
		getPageHeight(pageNum) {
			return this.pdfDoc.getPage(pageNum).then((page) => {
				const viewport = page.getViewport({ scale: 1.5 });
				return parseInt(viewport.height);
			});
		},

		// 设定参数，开始渲染canvas
		renderPage(num, canvasId) {
			this.pdfDoc.getPage(num).then((page) => {
				const canvas = document.getElementById(canvasId);
				const ctx = canvas.getContext("2d");
				const viewport = page.getViewport({ scale: 1.5 });
				canvas.height = viewport.height;
				canvas.width = viewport.width;
				const renderContext = {
					canvasContext: ctx,
					viewport: viewport,
				};
				page.render(renderContext);
			});
		},
		// 上一页
		onPrevPage() {
			if (this.pageNum > 1) {
				this.pageNum--;
				const pageTop = document.getElementById(
					`pdf-canvas-${this.pageNum}`
				).offsetTop;
				console.log('退回页面pageTop', pageTop)
				this.scrollTo(pageTop);
			}
		},

		// 下一页
		onNextPage() {
			console.log("执行了");
			if (this.pageNum < this.numPages) {
				this.pageNum++;
				const pageTop = document.getElementById(
					`pdf-canvas-${this.pageNum}`
				).offsetTop;
				console.log('下一个页面', pageTop)
				this.scrollTo(pageTop);
			}
		},

		// 滚动到指定页
		scrollTo(top) {
			// console.log("scrollTo()->top", top)
			// console.log("scrollTo()->this.pdfCanvasInner.scrollHeight", this.pdfCanvasInner.scrollHeight)
			// console.log("scrollTo()->this.pdfCanvasInner.clientHeight", this.pdfCanvasInner.clientHeight)
			const maxScrollTop =
				this.pdfCanvasInner.scrollHeight -
				this.pdfCanvasInner.clientHeight;
			if (top > maxScrollTop) {
				top = maxScrollTop;
			}
			const el = document.getElementById("pdfCanvasInner");
			el.scrollTop = top + 8;
		},

		// 滚动回调。滚动的高度除以页面高度 ，就能知道当前属于多少个页面了。 如果希望页面加间距， 那这里也要加间距的高度
		onScroll() {
			const scrollTop = document.getElementById("pdfCanvasInner").scrollTop;
			const currentPage = Math.ceil(scrollTop / this.pageHeight);
			this.pageNum = currentPage;
		},

		// 改变当前页码的回调
		pageChange(event) {
			console.log(event)
			const pageTop = document.getElementById(
				`pdf-canvas-${event}`
			).offsetTop;
			console.log('退回页面pageTop', pageTop)
			this.scrollTo(pageTop);

		},
	},
};
</script>
<style lang="scss">
body,
html {
	padding: 0px;
	margin: 0px;
	/* overflow: hidden; */
	height: 100%;
}

body>div#app {
	height: 100%;
}

body>div#app>div:first-child {
	height: 100%;
}
</style>

<style lang="scss" scoped>
.pdfCanvasWrappers {
	display: flex;
	width: 100%;
	height: 100%;
}

.pdfPreviewInputFrom {
	min-height: 100%;
	overflow-y: scroll;
	height: 100%;

	.title {
		font-size: 15px;
		font-weight: bold;
	}

	.tip {
		font-size: 13px;
		color: red;
	}

	.pdfPreviewInputFrom_inner {
		padding: 0px;
	}

	.pdfPreviewInputFrom_button {
		width: 100%;
		display: flex;
		align-items: center;
		height: 80px;
		padding: 20px 30px;
		box-sizing: border-box;
		margin-bottom: 30px;
		position: sticky;
		background: white;
		top: 0px;
		z-index: 4;
		box-shadow: 1px 1px 32px -16px #3e6ef1;
	}

	/deep/.pdfPreviewRuleFrom {

		padding: 0px 30px;

		.el-form-item__label {
			padding-bottom: 0px;
		}

		.el-form-item.el-form-item--mini {
			margin-bottom: 10px;
		}
	}
}

.pdfCanvasInner {
	min-height: 100%;
	width: 100%;
	display: flex;
	justify-content: flex-start;
	align-content: center;
	flex-direction: column;
	background-color: rgb(82, 84, 86);
	overflow-y: scroll;
	align-items: center;

	.pdfCanvas {
		margin: 60px 0px;
		margin-bottom: 180px;
		position: relative;
		box-shadow: 0px 2px 12px 4px hsla(210, 14%, 3%, 0.3);

		/* /deep/ .el-input__inner {
				background: none;
				border: none;
				width: 208px;
				height: 12px;
			} */
	}
}

/* 工具条 */
.pdfPreview_toolsBar {
	z-index: 2;
	height: 80px;
	position: sticky;
	display: flex;
	top: 0px;
	width: 100%;
	align-content: center;
	justify-content: center;
	background-color: #323639;
	box-shadow: 0px 6px 12px -4px #0607086b;
	color: #f1f1f1;

	>div {
		height: 80px;
		display: flex;
		align-items: center;
	}

	.buttonIcon {
		border-radius: 30px;
		color: #e7e7e7;
		font-size: 18px;
		padding: 15px;
		background: #353535;
		margin: 10px;
		transition: all 0.3s ease-out;

		&:hover {
			color: white;
			background: #464646;
		}
	}


	.buttonItem {
		display: flex;
		align-items: center;
		justify-content: center;
	}

	/* 跳转页面表单 */
	.pdfPreview_toolsBar-pageNum {
		display: inline-block;
		margin: 0px 10px;

		/deep/.el-input__inner {
			display: inline-block;
			border: none;
			line-height: 30px;
			height: 30px;
			text-align: center;
			padding: 0 2px;
			width: 40px;
			background: #1b1b1b;
			color: white;
		}

	}

}

#pdf-canvas {
	height: 100vh;
	width: auto;
	display: block;
	margin: 0 auto;
}
</style>