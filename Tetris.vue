<template>
	<div class="content">
		<div class="header">
			<text>{{score}}</text>
		</div>
		<div class="body" :style="{width: 750, height:gridSize*rowNumber}">
			<div class="next-preview" :style="{width:750-bodyWidth, height:750-bodyWidth}">
				<div class="game-row" v-for="(row, rowIndex) in previewGridList">
					<div class="grid-grid"
						 :style="{width:previewGridSize, height:previewGridSize}"
						 v-for="(grid, columnIndex) in row">
						<image class="grid-image"
							   :style="{opacity:grid.status!=gridStatus.empty?0.05:0.01}"
							   resize="contain"
							   :src="fullGridImage"></image>
					</div>
				</div>
			</div>
			<div class="game-body"
				 :style="{height:gridSize*rowNumber, width:bodyWidth}"
			>
				<div class="game-row" v-for="(row, rowIndex) in gridList">
					<div class="game-grid"
						 :style="{width:gridSize, height:gridSize}"
						 v-for="(grid, columnIndex) in row">
						<image class="grid-image"
							   :style="{opacity:grid.status!=gridStatus.empty?1:0.1}"
							   resize="contain"
							   :src="fullGridImage"></image>
					</div>
				</div>
			</div>
		</div>
		<div class="footer">
			<text class="reset-button" v-if="gameStatus!=0" @click="resetGame">{{gameStatus==1?'再玩一次':'重来'}}</text>
			<div class="control-content" v-if="gameStatus==0">
				<div class="control-row flex-center">
					<image class="control-image"
						   resize="contain"
						   :src="directionUp"
						   @touchend="touchEnd"
						   @longpress="longPress(controlDirection.up)"
						   @click="turn(controlDirection.up)"></image>
				</div>
				<div class="control-row flex-space-between">
					<image class="control-image"
						   resize="contain"
						   :src="directionLeft"
						   @touchend="touchEnd"
						   @longpress="longPress(controlDirection.left)"
						   @click="turn(controlDirection.left)"></image>
					<image class="control-image"
						   resize="contain"
						   :src="directionRight"
						   @touchend="touchEnd"
						   @longpress="longPress(controlDirection.right)"
						   @click="turn(controlDirection.right)"></image>
				</div>
				<div class="control-row flex-center">
					<image class="control-image"
						   resize="contain"
						   :src="directionDown"
						   @touchend="touchEnd"
						   @longpress="longPress(controlDirection.down)"
						   @click="turn(controlDirection.down)"></image>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
	const modal = weex.requireModule('modal')
	var timer = null
	var longPressTimer = null
	export default {
		computed: {
		    previewGridSize: function () {
				return (750-this.bodyWidth)/this.previewColumnNumber
			},
			gridSize: function () {
				return this.bodyWidth/this.columnNumber
			},
			bodyWidth: function () {
				return 650;
			}
		},
		data: {
			gameStatus: 0, // 0进行中 1赢 2输
			gridList: [],
			blockType: {
			    I: 0,
				J: 1,
				L: 2,
				O: 3,
				S: 4,
				Z: 5,
				T: 6
			},
			blockDirection: {
			    normal: 0,  // 默认
				left: 1,    // 逆时针旋转90度
				down: 2,    // 旋转180度
				right: 3,   // 逆时针针旋转270度
			},
			currentBlock: null,
			removingCount: 0, // 当前连续消除第几行
			score: 0, // 得分
			nextBlock: null,
			rowNumber: 20,
			columnNumber: 18,
			previewGridList:[],
			previewRowNumber:4,
			previewColumnNumber: 4,
			gridStatus: {
				empty: 0,
				full: 1
			},
			controlDirection: {
				left: 1,
				up: 2,
				right: -1,
				down: -2
			},
			headerHighlight: false,
			foodHighlight: false,
			speedLevel: 0,
			turning: false,
			speeds: [800, 700, 600, 500, 400, 300, 200, 100, 50],
			fullGridImage: require('@/assets/img/snake_header_1.png'),
			directionUp: require('@/assets/img/direction_up.png'),
			directionLeft: require('@/assets/img/direction_left.png'),
			directionRight: require('@/assets/img/direction_right.png'),
			directionDown: require('@/assets/img/direction_down.png'),
		},
		methods: {
			turn: function (direction) {
				if (this.turning || this.gameStatus!=0) {
					return
				}
				this.turning = true
				switch (direction) {
					case this.controlDirection.left:
						this.currentBlock.newOffsetX--
					    break
					case this.controlDirection.up:
						this.currentBlock.direction++
						if (this.currentBlock.direction>3) {
						    this.currentBlock.direction = this.blockDirection.normal
						}
					    break
					case this.controlDirection.right:
						this.currentBlock.newOffsetX++
						break
					case this.controlDirection.down:
						this.currentBlock.newOffsetY++
						break
				}
				this.updateControl(direction)
			},
			touchEnd: function () {
				clearInterval(longPressTimer)
				longPressTimer = null
			},
			longPress: function (direction) {
			    if (this.speeds[this.speedLevel] <= 100) {
			        return
				}
			    var self = this
				longPressTimer = setInterval(function () {
					self.turn(direction)
				}, 100)
			},
			gridImage: function (grid) {
				switch (grid.status) {
					case this.gridStatus.empty:
					    break
					case this.gridStatus.full:
					    break
					default:
						return ' '
				}
			},
			failed: function () {
				this.gameStatus = 2
				this.stopWalk()
				this.touchEnd()
				toast('你输了！', 3)
			},
			success: function () {
				this.gameStatus = 1
				this.stopWalk()
				this.touchEnd()
				toast('你赢了！', 3)
			},
			// 消除行
			removeRow: function (rowIndex) {
			    var gridRow = this.gridList[rowIndex]
				for (var i in gridRow) {
			        gridRow[i].status = this.gridStatus.empty
				}
				this.score += (100 + this.removingCount*100)
				this.removingCount++
			    this.gridList.splice(rowIndex, 1, gridRow)
				var self = this
				setTimeout(function () {
				    var lastRow = gridRow
				    for (var i=rowIndex-1; i>=0; i--) {
				        var row = self.gridList[i]
						var fullCount = 0
						for (var j=0; j<row.length; j++) {
				            if (row[j].status == self.gridStatus.full) {
								fullCount++
							}
							lastRow[j].status = row[j].status
						}
						if (fullCount==0) {
				            break
						}
						lastRow = row
					}
					setTimeout(self.checkRemove, 100)
				}, 80)
			},
			// 检查是否能消除行
			checkRemove: function () {
				for (var i=this.rowNumber-1; i>=0; i--) {
				    var fullCount = 0
				    for (var j=0; j<this.columnNumber; j++) {
						if (this.gridList[i][j].status == this.gridStatus.full) {
						    fullCount++
						}
					}
					if (fullCount == this.columnNumber) {
						this.removeRow(i)
						return
					} else if (fullCount == 0) {
				        break
					} else if (i == 0) {
						this.failed()
						return
					}
				}
				this.turning = false
				this.removingCount = 0
				this.startWalk()
			},
			// 检查点是否合法，用来防止移出边界或与其它方块重合
			checkPoints: function (points, oldPoints, controlDirection, targetDirection) {
			    // 这里可以优化：如果是旋转，扩大检查范围，看旋转经过路径是否有障碍物
//			    if (controlDirection == this.controlDirection.up) {
//					switch (targetDirection) {
//
//					}
//				}
				for (var i in points) {
					if (points[i].row<0) {
						continue
					}
					if (points[i].row>this.rowNumber-1
						|| points[i].column<0
						|| points[i].column>this.columnNumber-1
						|| (this.gridList[points[i].row][points[i].column].status==this.gridStatus.full && !containsPoint(oldPoints, points[i]))) {
						return false
					}
				}
				return true
			},
			// 更新控制
			updateControl: function (direction) {
			    var newPoints = this.getBlockPoints(this.currentBlock)
				if (direction != this.controlDirection.down) {
					newPoints = this.formatPoint(newPoints)
				}
				if (this.checkPoints(newPoints, this.currentBlock.points, direction, this.currentBlock.direction)) {
					this.currentBlock.offsetX = this.currentBlock.newOffsetX
					this.currentBlock.offsetY = this.currentBlock.newOffsetY
					this.currentBlock.points = newPoints
					this.updateGrid(this.currentBlock)
				} else {
					this.currentBlock.newOffsetX = this.currentBlock.offsetX
					this.currentBlock.newOffsetY = this.currentBlock.offsetY
					if (direction == this.controlDirection.up) {
						this.currentBlock.direction--
						if (this.currentBlock.direction<0) {
							this.currentBlock.direction = this.blockDirection.right
						}
					}
					if (!this.blockCanDrop(this.currentBlock)) {
						this.currentBlock = this.nextBlock
						this.nextBlock = this.randomBlock(this.nextBlock.type)
						this.updatePreviewGrid()
						this.stopWalk()
						this.checkRemove()
						return
					}
				}
				this.turning = false
			},
			// 方块是否还能下落
			blockCanDrop: function (block) {
			    block.newOffsetY = block.offsetY+1
				var newPoints = this.getBlockPoints(block)
				var canDrop = this.checkPoints(newPoints, block.points, this.controlDirection.down, this.currentBlock.direction)
				block.newOffsetY = block.offsetY
				return canDrop
			},
			// 自动下落
			drop: function () {
				this.currentBlock.newOffsetY++
				this.updateControl(this.controlDirection.down)
			},
			startWalk: function () {
			    this.updateGrid(this.currentBlock)
				this.speedLevel += this.score%((this.speedLevel+1)*2000)
				if (this.speedLevel>=this.speeds.length) {
					this.speedLevel = this.speeds.length - 1
				}
				timer = setInterval(this.drop, this.speeds[this.speedLevel])
			},
			stopWalk: function () {
				clearInterval(timer)
				timer = null
			},
			getGrid: function (block) {
			    var grids = []
				for (var i in block.points) {
			        var point = block.points[i]
					if (point.row>=this.rowNumber
						|| point.row<0
						|| point.column>=this.columnNumber
						|| point.column<0) {
			            continue
					}
					grids.push(this.gridList[point.row][point.column])
				}
				return grids
			},
			// 更新方块
			updateGrid: function (block) {
				var newGrids = this.getGrid(block)
				for (var i in block.grids) {
				    var oldGrid = block.grids[i]
					oldGrid.status = this.gridStatus.empty
				}
				for (var i in newGrids) {
					var newGrid = newGrids[i]
					newGrid.status = this.gridStatus.full
				}
				block.grids = newGrids
				if (newGrids.length > 0) {
				    var firstGrid = newGrids[0]
					this.gridList[firstGrid.row].splice(firstGrid.column, 1, firstGrid)
				}
			},
			// 更新预览
			updatePreviewGrid: function () {
				var points = this.getBlockPoints(this.nextBlock)
				var offsetRow = 0
				var offsetColumn = 0
				for (var i in points) {
					if (points[i].row < 0 && 0-points[i].row > offsetRow) {
						offsetRow = 0-points[i].row
					} else if (points[i].row > this.previewRowNumber-1 && this.previewRowNumber-1-points[i].row < offsetRow) {
						offsetRow = this.previewRowNumber-1-points[i].row
					}
					if (points[i].column < 0 && 0-points[i].column > offsetColumn) {
						offsetColumn = 0-points[i].column
					} else if (points[i].column > this.previewColumnNumber-1 && this.previewColumnNumber-1-points[i].column < offsetColumn) {
						offsetColumn = this.previewColumnNumber-1-points[i].column
					}
				}
				for (var i=0; i<this.previewRowNumber; i++) {
					for (var j=0; j<this.previewColumnNumber; j++) {
						this.previewGridList[i][j].status = this.gridStatus.empty
					}
				}
				for (var i in points) {
					points[i].row = points[i].row+offsetRow
					points[i].column = points[i].column+offsetColumn
					this.previewGridList[points[i].row][points[i].column].status = this.gridStatus.full
				}
			},
			// 获取方块的点
			getBlockPoints: function (block) {
			    var points = []
				switch (block.type) {
					case this.blockType.Z: {
					    switch (block.direction) {
							case this.blockDirection.normal:
							case this.blockDirection.down:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
							case this.blockDirection.right:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									}
								]
								break
						}
					}
						break
					case this.blockType.S: {
						switch (block.direction) {
							case this.blockDirection.normal:
							case this.blockDirection.down:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber / 2 + 1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber / 2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber / 2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber / 2 - 1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
							case this.blockDirection.right:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber / 2 - 1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber / 2 - 1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber / 2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber / 2 + block.newOffsetX
									}
								]
								break
						}
					}
						break
					case this.blockType.O: {
						points = [
							{
								row: -2 + block.newOffsetY,
								column: this.columnNumber/2-1 + block.newOffsetX
							},
							{
								row: -2 + block.newOffsetY,
								column: this.columnNumber/2 + block.newOffsetX
							},
							{
								row: -1 + block.newOffsetY,
								column: this.columnNumber/2-1 + block.newOffsetX
							},
							{
								row: -1 + block.newOffsetY,
								column: this.columnNumber/2 + block.newOffsetX
							}
						]
					}
						break
					case this.blockType.L: {
						switch (block.direction) {
							case this.blockDirection.normal:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.down:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.right:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									}
								]
								break
						}
					}
						break
					case this.blockType.I: {
						switch (block.direction) {
							case this.blockDirection.normal:
							case this.blockDirection.down:
								points = [
									{
										row: -4 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
							case this.blockDirection.right:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
						}
					}
						break
					case this.blockType.J: {
						switch (block.direction) {
							case this.blockDirection.normal:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.down:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.right:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
						}
					}
						break
					case this.blockType.T: {
						switch (block.direction) {
							case this.blockDirection.normal:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.down:
								points = [
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.left:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -1 + block.newOffsetY,
										column: this.columnNumber/2-1 + block.newOffsetX
									}
								]
								break
							case this.blockDirection.right:
								points = [
									{
										row: -3 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2 + block.newOffsetX
									},
									{
										row: -2 + block.newOffsetY,
										column: this.columnNumber/2+1 + block.newOffsetX
									}
								]
								break
						}

					}
						break
				}
				return points
			},
			// 矫正方块
			formatPoint: function (points) {
				var offsetRow = 0
				var offsetColumn = 0
				for (var i in points) {
					if (points[i].row > this.rowNumber-1 && this.rowNumber-1-points[i].row < offsetRow) {
						offsetRow = this.rowNumber-1-points[i].row
					}
					if (points[i].column < 0 && 0-points[i].column > offsetColumn) {
						offsetColumn = 0-points[i].column
					} else if (points[i].column > this.columnNumber-1 && this.columnNumber-1-points[i].column < offsetColumn) {
						offsetColumn = this.columnNumber-1-points[i].column
					}
				}
				if (offsetRow!=0 || offsetColumn!=0) {
					for (var i in points) {
						points[i].row = points[i].row+offsetRow
						points[i].column = points[i].column+offsetColumn
					}

					this.currentBlock.newOffsetX += offsetColumn
					this.currentBlock.newOffsetY += offsetRow
				}
				return points
			},
			// 随机方块，随机旋转
			randomBlock: function (excludeType) {
				var blockKeys = Object.keys(this.blockType)
				var type = this.blockType[blockKeys[Math.floor(Math.random()*blockKeys.length)]]
				if (type == excludeType) {
				    return this.randomBlock()
				}
				var directionKeys = Object.keys(this.blockDirection)
				var block = {
					type: type,
					offsetX: 0,
					offsetY: 0,
					newOffsetX: 0,
					newOffsetY: 0,
					direction: this.blockDirection[directionKeys[Math.floor(Math.random()*directionKeys.length)]],
					grids: [],
					points: {}
				}
				block.points = this.getBlockPoints(block)
				return block
			},
			resetGame: function () {
				this.stopWalk()
				this.gameStatus = 0
				this.speedLevel = 0
				this.removingCount = 0
				this.score = 0
				// 游戏区
				for (var i=0; i<this.rowNumber; i++) {
					var row
					if (i<this.gridList.length) {
						row = this.gridList[i]
						this.gridList.splice(i, 1, row)
					} else  {
						row = []
						this.gridList.push(row)
					}
					for (var j=0; j<this.columnNumber; j++) {
						var grid
						if (j<row.length) {
							grid = row[j]
						} else {
							grid = {}
							row.push(grid)
						}
						grid.row = i
						grid.column = j
						grid.status = this.gridStatus.empty
					}
				}
				// 预览区
				for (var i=0; i<this.previewRowNumber; i++) {
					var row
					if (i<this.previewGridList.length) {
						row = this.previewGridList[i]
						this.previewGridList.splice(i, 1, row)
					} else  {
						row = []
						this.previewGridList.push(row)
					}
					for (var j=0; j<this.previewColumnNumber; j++) {
						var grid
						if (j<row.length) {
							grid = row[j]
						} else {
							grid = {}
							row.push(grid)
						}
						grid.row = i
						grid.column = j
						grid.status = this.gridStatus.empty
					}
				}
				this.currentBlock = this.randomBlock()
				this.nextBlock = this.randomBlock(this.currentBlock.type)
				this.updatePreviewGrid()
				this.startWalk()
			}
		},
		created: function () {
			this.resetGame()
		},
		beforeDestroy: function () {
			this.stopWalk()
		}
	}
	function containsPoint(points, point) {
		for (var i in points) {
		    if (points[i].row==point.row&&points[i].column==point.column) {
		        return true
			}
		}
		return false
	}
	function toast(text, duration) {
		modal.toast({
			message: text,
			duration: duration
		})
	}
</script>

<style scoped>
	.content {
		background-color: white;
		flex-direction: column;
		align-items: center;
	}
	.header {
		height: 50px;
	}
	.body {
		flex-direction: row;
		background-color: oldlace;
	}
	.footer {
		flex: 1;
		flex-direction: row;
		justify-content: center;
		align-items: center;
	}
	.next-preview {
		flex-direction: column;
	}
	.game-body {
		flex-direction: column;
	}
	.game-row {
		flex-direction: row;
	}
	.game-grid {
		flex-direction: row;
		justify-content: center;
		align-items: center;
	}
	.grid-image {
		background-color: rgba(0,0,0,0);
		position: absolute;
		left: 0;
		top: 0;
		right: 0;
		bottom: 0;
	}
	.reset-button {
		font-size: 30wx;
		color: orange;
	}
	.control-content {
		width: 400px;
		height: 400px;
		flex-direction: column;
	}
	.control-row {
		flex: 1;
		flex-direction: row;
		align-items: center;
	}
	.control-image {
		width: 120px;
		height: 120px;
	}
	.flex-center {
		justify-content: center;
	}
	.flex-space-between {
		justify-content: space-between;
	}
</style>
