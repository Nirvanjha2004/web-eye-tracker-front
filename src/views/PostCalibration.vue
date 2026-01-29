<template>
    <div class="scroll-container">
        <canvas id="canvas" style="width: 100%; height: 100%;" />

        <!-- Recalibration Overlay -->
        <div v-if="isRecalibrating" class="recalibration-overlay">
            <div class="recalibration-instructions">
                <v-icon size="48" color="#FF425A" class="mb-3">mdi-target</v-icon>
                <h3>Recalibrating Point {{ recalibrationPoint + 1 }}</h3>
                <p>Look at the red cross and press <strong>S</strong> when ready</p>
                <p>Keep your head still and follow with your eyes only</p>
                <div class="recalibration-buttons">
                    <v-btn color="#FF425A" dark @click="cancelRecalibration">
                        Cancel Recalibration
                    </v-btn>
                </div>
            </div>
        </div>

        <PointModal
            :x="Number(x)"
            :y="Number(y)"
            :precision="Number(precision)"
            :accuracy="Number(accuracy)"
            :dialog="dialog"
            :pointNumber="pointNumber"
            @close="dialogCancel"
            @select="select"
            @recalibratePoint="startPointRecalibration"
        />

        <ConfigModal
            :configDialog="configDialog"
            @close="configDialogCancel"
            @recalib="recalibrate"
            @save="saveCalib"
        />

        <v-col class="pa-0">
            <DraggableFloatingButton
                @click="callConfigModal"
                icon="mdi-cog"
            />
        </v-col>

        <div v-if="redirectingToRuxailab" class="button-overlay">
            <v-btn @click="recalibrate" color="primary">Recalibrate</v-btn>
            <v-btn @click="sendCalibToRuxailab" color="success">
                Send this calib to Ruxailab
            </v-btn>
        </div>
    </div>
</template>

<script>
import PointModal from '@/components/calibration/PointModal.vue'
import DraggableFloatingButton from '@/components/general/DraggableFloatingButton.vue'
import ConfigModal from '@/components/calibration/ConfigModal.vue'

export default {
    components: {
        PointModal,
        DraggableFloatingButton,
        ConfigModal
    },

    data() {
        return {
            x: 0,
            y: 0,
            precision: 0,
            accuracy: 0,
            dialog: false,
            configDialog: false,
            pointNumber: 0,
            redirectingToRuxailab: false,
            isRecalibrating: false,
            recalibrationPoint: null
        }
    },

    async mounted() {
        await this.verifyFromRuxailab()
        this.applyCalibResult()
        this.initCanvas()
        this.drawCalibPoints()
        this.addCanvasClickListener()
    },

    computed: {
        radius() {
            return this.$store.state.calibration.radius
        },
        backgroundColor() {
            return this.$store.state.calibration.backgroundColor
        },
        pattern() {
            return this.$store.state.calibration.runtime.usedPattern
        },
        mockPattern() {
            return this.$store.state.calibration.mockPattern
        },
        threshold() {
            return this.$store.state.calibration.threshold
        },
        fromDashboard() {
            return this.$store.state.calibration.fromDashboard
        },
        calibValidationResult() {
            return this.$store.state.predict.calibValidationResult
        },
        circleIrisPoints() {
            return this.$store.state.calibration.runtime.circleIrisPoints
        },
        calibPredictionPoints() {
            return this.$store.state.calibration.runtime.calibPredictionPoints
        }
    },

    watch: {
        threshold() {
            this.drawCalibPoints()
        },
        mockPattern() {
            this.drawCalibPoints()
        }
    },

    methods: {
        applyCalibResult() {
            if (!this.calibValidationResult) {
                console.warn('❌ calibValidationResult é null/undefined')
                return
            }

            console.log('✅ calibValidationResult encontrado:', this.calibValidationResult)

            const adapted = this.pattern.map((p, idx) => {
                const xKey = Math.round(p.x).toString()
                const yKey = Math.round(p.y).toString()

                console.log(`📍 Ponto ${idx}: x=${p.x}, y=${p.y} -> keys: ${xKey}, ${yKey}`)

                const data = this.calibValidationResult?.[xKey]?.[yKey]

                if (!data) {
                    console.warn(`⚠️ Nenhum dado para ponto ${idx} (${xKey}, ${yKey})`)
                    return {
                        ...p,
                        predictionX: [],
                        predictionY: [],
                        precision: null,
                        accuracy: null
                    }
                }

                console.log(`✅ Dados encontrados para ponto ${idx}:`, data)
                return {
                    ...p,
                    predictionX: data.predicted_x || [],
                    predictionY: data.predicted_y || [],
                    precision: data.PrecisionSD,
                    accuracy: data.Accuracy
                }
            })

            this.$store.commit('setCalibrationPattern', adapted)
        },

        async verifyFromRuxailab() {
            const params = new URLSearchParams(window.location.search)
            this.redirectingToRuxailab = params.has('redirectingToRuxailab')
        },

        callConfigModal() {
            this.configDialog = true
        },

        recalibrate() {
            this.$router.back()
        },

        async sendCalibToRuxailab() {
            const screenHeight = window.screen.height
            const screenWidth = window.screen.width

            await this.$store.dispatch('sendData', {
                fromRuxailab: true,
                circleIrisPoints: this.circleIrisPoints,
                calibPredictionPoints: this.calibPredictionPoints,
                screenHeight,
                screenWidth,
                k: this.$store.state.calibration.pointNumber,
                threshold: this.threshold
            })
        },

        select(pointNumber) {
            this.$store.commit(
                'setMockPatternElement',
                this.pattern[pointNumber]
            )
        },

        drawCalibPoints() {
            console.log('🎨 Iniciando drawCalibPoints')
            console.log('📊 Pattern:', this.pattern)
            console.log('📈 Threshold:', this.threshold)

            this.initCanvas()
            const pointSize = 3.5

            this.pattern.forEach((p, idx) => {
                console.log(`\n🔵 Desenhando ponto ${idx} em (${p.x}, ${p.y})`)
                console.log(`   predictionX: ${p.predictionX?.length || 0} valores`)
                console.log(`   predictionY: ${p.predictionY?.length || 0} valores`)

                this.drawCalibMarks(p.x, p.y, 30, 'grey')

                let sumX = 0
                let sumY = 0
                let count = 0

                p.predictionX?.forEach((px, i) => {
                    const py = p.predictionY[i]
                    const dist = this.euclidianDistance(p.x, px, p.y, py)

                    console.log(`   [${i}] px=${px}, py=${py}, dist=${dist.toFixed(2)}, threshold=${this.threshold}, pass=${dist <= this.threshold}`)

                    this.drawPoints(px, py, pointSize, dist <= this.threshold ? 'green' : 'grey')

                    if (dist <= this.threshold) {
                        sumX += px
                        sumY += py
                        count++
                    }
                })

                console.log(`   ✏️ Total pontos dentro threshold: ${count}`)

                if (count > 0) {
                    const cx = sumX / count
                    const cy = sumY / count
                    console.log(`   🎯 Centróide em (${cx.toFixed(2)}, ${cy.toFixed(2)})`)
                    this.drawDash(cx, cy, p.x, p.y, 'red')
                    this.drawCentroid(cx, cy, 1 + (p.precision || 0) * 25, 'rgba(0,0,255,0.3)')
                }
            })
        },

        euclidianDistance(x0, x1, y0, y1) {
            return Math.hypot(x1 - x0, y1 - y0)
        },

        initCanvas() {
            const canvas = document.getElementById('canvas')
            console.log('🖼️ Canvas:', canvas)
            canvas.width = window.innerWidth
            canvas.height = window.innerHeight

            const ctx = canvas.getContext('2d')
            ctx.clearRect(0, 0, canvas.width, canvas.height)
            ctx.fillStyle = this.backgroundColor
            ctx.fillRect(0, 0, canvas.width, canvas.height)
        },

        drawCentroid(x, y, r, color) {
            const ctx = document.getElementById('canvas').getContext('2d')
            ctx.fillStyle = color
            ctx.beginPath()
            ctx.arc(x, y, r, 0, Math.PI * 2)
            ctx.fill()
        },

        drawPoints(x, y, r, color) {
            const ctx = document.getElementById('canvas').getContext('2d')
            ctx.fillStyle = color
            ctx.beginPath()
            ctx.arc(x, y, r, 0, Math.PI * 2)
            ctx.fill()
        },

        drawCalibMarks(x, y, s, color) {
            const ctx = document.getElementById('canvas').getContext('2d')
            ctx.strokeStyle = color

            ctx.beginPath()
            ctx.moveTo(x - s, y)
            ctx.lineTo(x + s, y)
            ctx.stroke()

            ctx.beginPath()
            ctx.moveTo(x, y - s)
            ctx.lineTo(x, y + s)
            ctx.stroke()
        },

        drawDash(fx, fy, tx, ty, color) {
            const ctx = document.getElementById('canvas').getContext('2d')
            ctx.strokeStyle = color
            ctx.setLineDash([5, 5])
            ctx.beginPath()
            ctx.moveTo(fx, fy)
            ctx.lineTo(tx, ty)
            ctx.stroke()
            ctx.setLineDash([])
        },

        dialogCancel(v) {
            this.dialog = v
        },

        configDialogCancel(v) {
            this.configDialog = v
        },

        async saveCalib() {
            await this.$store.dispatch('saveCalib')
            this.$router.push('/dashboard')
        },

        addCanvasClickListener() {
            const canvas = document.getElementById('canvas')
            canvas.addEventListener('click', this.handleCanvasClick)
        },

        handleCanvasClick(event) {
            if (this.isRecalibrating) return

            const canvas = document.getElementById('canvas')
            const rect = canvas.getBoundingClientRect()
            const clickX = event.clientX - rect.left
            const clickY = event.clientY - rect.top

            // Find the closest calibration point
            let closestPoint = null
            let closestDistance = Infinity
            let closestIndex = -1

            this.pattern.forEach((point, index) => {
                const distance = this.euclidianDistance(clickX, point.x, clickY, point.y)
                if (distance < closestDistance && distance < 50) { // 50px threshold for click detection
                    closestDistance = distance
                    closestPoint = point
                    closestIndex = index
                }
            })

            if (closestPoint) {
                this.showPointModal(closestPoint, closestIndex)
            }
        },

        showPointModal(point, index) {
            this.x = Math.round(point.x)
            this.y = Math.round(point.y)
            this.precision = point.precision || 0
            this.accuracy = point.accuracy || 0
            this.pointNumber = index
            this.dialog = true
        },

        async startPointRecalibration(pointIndex) {
            console.log(`Starting recalibration for point ${pointIndex}`)
            
            // Check if we have the necessary components
            if (!this.$store.state.detect?.model) {
                this.showErrorMessage('Face detection model not available. Please restart the calibration process.')
                return
            }
            
            this.isRecalibrating = true
            this.recalibrationPoint = pointIndex
            
            try {
                // Show recalibration UI
                this.showRecalibrationOverlay(pointIndex)
            } catch (error) {
                console.error('Failed to start recalibration:', error)
                this.showErrorMessage('Failed to start recalibration. Please try again.')
                this.cancelRecalibration()
            }
        },

        showRecalibrationOverlay(pointIndex) {
            const point = this.pattern[pointIndex]
            
            // Clear canvas and show only the point to recalibrate
            this.initCanvas()
            this.drawCalibMarks(point.x, point.y, 30, 'red')
            
            // Show instructions overlay
            this.showRecalibrationInstructions(pointIndex)
        },

        showRecalibrationInstructions(pointIndex) {
            // Remove any existing overlay
            const existingOverlay = document.getElementById('recalibration-overlay')
            if (existingOverlay) {
                document.body.removeChild(existingOverlay)
            }

            // Create overlay with instructions
            const overlay = document.createElement('div')
            overlay.id = 'recalibration-instructions-overlay'
            overlay.style.cssText = `
                position: fixed;
                top: 50%;
                left: 50%;
                transform: translate(-50%, -50%);
                background: rgba(255, 255, 255, 0.95);
                padding: 20px;
                border-radius: 10px;
                box-shadow: 0 4px 20px rgba(0,0,0,0.3);
                z-index: 1000;
                text-align: center;
                max-width: 400px;
            `
            
            overlay.innerHTML = `
                <h3>Recalibrating Point ${pointIndex + 1}</h3>
                <p>Look at the red cross and press <strong>S</strong> when ready</p>
                <p>Keep your head still and follow with your eyes only</p>
                <button id="start-recalib-btn" style="
                    background: #FF425A;
                    color: white;
                    border: none;
                    padding: 10px 20px;
                    border-radius: 5px;
                    cursor: pointer;
                    margin: 5px;
                ">Start Recalibration</button>
                <button id="cancel-recalib-btn" style="
                    background: #666;
                    color: white;
                    border: none;
                    padding: 10px 20px;
                    border-radius: 5px;
                    cursor: pointer;
                    margin: 5px;
                ">Cancel</button>
            `
            
            document.body.appendChild(overlay)
            
            // Add event listeners
            const startBtn = document.getElementById('start-recalib-btn')
            const cancelBtn = document.getElementById('cancel-recalib-btn')
            
            if (startBtn) {
                startBtn.addEventListener('click', () => {
                    this.executePointRecalibration(pointIndex)
                    if (document.body.contains(overlay)) {
                        document.body.removeChild(overlay)
                    }
                })
            }
            
            if (cancelBtn) {
                cancelBtn.addEventListener('click', () => {
                    this.cancelRecalibration()
                    if (document.body.contains(overlay)) {
                        document.body.removeChild(overlay)
                    }
                })
            }
        },

        async executePointRecalibration(pointIndex) {
            const point = this.pattern[pointIndex]
            
            try {
                // Show the point to recalibrate
                this.drawPoint(point.x, point.y, 1)
                
                // Start data collection for this point
                await this.recalibratePoint(point, pointIndex)
                
                // Update the display
                this.drawCalibPoints()
                
                this.isRecalibrating = false
                this.recalibrationPoint = null
                
                // Show success message
                this.showRecalibrationSuccess(pointIndex)
                
            } catch (error) {
                console.error('Recalibration failed:', error)
                this.cancelRecalibration()
            }
        },

        async recalibratePoint(point, pointIndex) {
            // Clear existing data for this point
            point.data = []
            
            // Use the same extraction logic as the main calibration
            const msPerCapture = this.$store.state.calibration.msPerCapture
            const predByPointCount = this.$store.state.calibration.samplePerPoint
            
            return new Promise((resolve, reject) => {
                // let sampleCount = 0
                
                const keyHandler = async (event) => {
                    if (event.key === 's' || event.key === 'S' || event.key === 'Enter') {
                        document.removeEventListener('keydown', keyHandler)
                        
                        try {
                            // Collect samples for this point
                            for (let i = 0; i < predByPointCount; i++) {
                                const prediction = await this.detectFace()
                                
                                if (!prediction || prediction.length === 0) {
                                    console.warn('Face not detected during recalibration')
                                    await new Promise(resolve => setTimeout(resolve, 500))
                                    continue
                                }
                                
                                const pred = prediction[0]
                                if (!pred.annotations?.leftEyeIris || !pred.annotations?.rightEyeIris) {
                                    console.warn('Incomplete face landmarks during recalibration')
                                    await new Promise(resolve => setTimeout(resolve, 500))
                                    continue
                                }
                                
                                // Check for blinks
                                const leftIris = pred.annotations.leftEyeIris
                                const rightIris = pred.annotations.rightEyeIris
                                const leftEyelid = pred.annotations.leftEyeUpper0.concat(pred.annotations.leftEyeLower0)
                                const rightEyelid = pred.annotations.rightEyeUpper0.concat(pred.annotations.rightEyeLower0)
                                
                                const isLeftBlink = this.calculateDistance(leftEyelid[3], leftEyelid[11]) < this.$store.state.calibration.leftEyeTreshold
                                const isRightBlink = this.calculateDistance(rightEyelid[3], rightEyelid[11]) < this.$store.state.calibration.rightEyeTreshold
                                
                                if (!isLeftBlink && !isRightBlink) {
                                    const newPrediction = { 
                                        leftIris: leftIris[0], 
                                        rightIris: rightIris[0] 
                                    }
                                    point.data.push(newPrediction)
                                    
                                    // Update visual feedback
                                    const radius = (this.$store.state.calibration.radius / predByPointCount) * i
                                    this.drawPoint(point.x, point.y, radius)
                                    
                                    // sampleCount++
                                }
                                
                                await new Promise(resolve => setTimeout(resolve, msPerCapture))
                            }
                            
                            // Update the calibration data
                            await this.updatePointCalibration(point, pointIndex)
                            resolve()
                            
                        } catch (error) {
                            reject(error)
                        }
                    }
                }
                
                document.addEventListener('keydown', keyHandler)
            })
        },

        async updatePointCalibration(point, pointIndex) {
            // Update the training data
            const newTrainingData = []
            point.data.forEach(element => {
                newTrainingData.push({
                    left_iris_x: element.leftIris[0],
                    left_iris_y: element.leftIris[1],
                    right_iris_x: element.rightIris[0],
                    right_iris_y: element.rightIris[1],
                    point_x: point.x,
                    point_y: point.y,
                })
            })
            
            // Replace the data for this specific point in the training set
            const circleIrisPoints = [...this.circleIrisPoints]
            const pointsPerCalibPoint = this.$store.state.calibration.samplePerPoint
            const startIndex = pointIndex * pointsPerCalibPoint
            // const endIndex = startIndex + pointsPerCalibPoint
            
            // Remove old data for this point
            circleIrisPoints.splice(startIndex, pointsPerCalibPoint, ...newTrainingData)
            
            // Update store
            this.$store.commit('setRuntimeData', {
                ...this.$store.state.calibration.runtime,
                circleIrisPoints
            })
            
            // Recalculate predictions for this point
            await this.recalculatePointPredictions(pointIndex)
        },

        async recalculatePointPredictions(pointIndex) {
            // Send updated data to backend for recalculation
            const screenHeight = window.innerHeight
            const screenWidth = window.innerWidth
            
            let predictions = await this.$store.dispatch('sendData', {
                fromRuxailab: false,
                circleIrisPoints: this.circleIrisPoints,
                calibPredictionPoints: this.calibPredictionPoints,
                screenHeight,
                screenWidth,
                k: this.$store.state.calibration.pointNumber,
                threshold: this.$store.state.calibration.threshold
            })
            
            if (typeof predictions === 'string') {
                predictions = predictions.replace(/NaN/g, '1')
                predictions = JSON.parse(predictions)
            }
            
            // Update only the recalibrated point
            const point = this.pattern[pointIndex]
            const element = predictions[point.x.toString().split('.')[0]][point.y.toString().split('.')[0]]
            
            point.precision = element.PrecisionSD.toFixed(2)
            point.accuracy = element.Accuracy.toFixed(2)
            point.predictionX = element.predicted_x
            point.predictionY = element.predicted_y
            
            // Update store
            this.$store.commit('setPattern', this.pattern)
        },

        showRecalibrationSuccess(pointIndex) {
            const overlay = document.createElement('div')
            overlay.style.cssText = `
                position: fixed;
                top: 50%;
                left: 50%;
                transform: translate(-50%, -50%);
                background: rgba(76, 175, 80, 0.95);
                color: white;
                padding: 20px;
                border-radius: 10px;
                z-index: 1000;
                text-align: center;
            `
            
            overlay.innerHTML = `
                <h3>✓ Point ${pointIndex + 1} Recalibrated Successfully!</h3>
                <p>New accuracy and precision values have been calculated</p>
            `
            
            document.body.appendChild(overlay)
            
            setTimeout(() => {
                if (document.body.contains(overlay)) {
                    document.body.removeChild(overlay)
                }
            }, 3000)
        },

        showErrorMessage(message) {
            const overlay = document.createElement('div')
            overlay.style.cssText = `
                position: fixed;
                top: 50%;
                left: 50%;
                transform: translate(-50%, -50%);
                background: rgba(244, 67, 54, 0.95);
                color: white;
                padding: 20px;
                border-radius: 10px;
                z-index: 1000;
                text-align: center;
                max-width: 400px;
            `
            
            overlay.innerHTML = `
                <h3>⚠ Recalibration Error</h3>
                <p>${message}</p>
                <button onclick="this.parentElement.remove()" style="
                    background: white;
                    color: #f44336;
                    border: none;
                    padding: 8px 16px;
                    border-radius: 4px;
                    cursor: pointer;
                    margin-top: 10px;
                ">OK</button>
            `
            
            document.body.appendChild(overlay)
            
            setTimeout(() => {
                if (document.body.contains(overlay)) {
                    document.body.removeChild(overlay)
                }
            }, 5000)
        },

        cancelRecalibration() {
            this.isRecalibrating = false
            this.recalibrationPoint = null
            this.drawCalibPoints()
        },

        calculateDistance(eyelidTip, eyelidBottom) {
            const xDistance = eyelidBottom[0] - eyelidTip[0]
            const yDistance = eyelidBottom[1] - eyelidTip[1]
            return Math.sqrt(xDistance * xDistance + yDistance * yDistance)
        },

        async detectFace() {
            // Check if face detection model is available
            const model = this.$store.state.detect?.model
            if (!model) {
                console.error('Face detection model not available for recalibration')
                throw new Error('Face detection model not available. Please restart the calibration process.')
            }
            
            // Check if video element exists
            let videoElement = document.getElementById('video-tag')
            if (!videoElement) {
                console.warn('Video element not found, attempting to restart camera...')
                await this.startWebCamForRecalibration()
                videoElement = document.getElementById('video-tag')
                if (!videoElement) {
                    throw new Error('Unable to access camera for recalibration')
                }
            }
            
            try {
                return await model.estimateFaces({ input: videoElement })
            } catch (error) {
                console.error('Face detection failed:', error)
                throw new Error('Face detection failed during recalibration')
            }
        },

        async startWebCamForRecalibration() {
            // Create video element if it doesn't exist
            let videoElement = document.getElementById('video-tag')
            if (!videoElement) {
                videoElement = document.createElement('video')
                videoElement.id = 'video-tag'
                videoElement.autoplay = true
                videoElement.style.display = 'none'
                document.body.appendChild(videoElement)
            }

            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    audio: false,
                    video: true
                })
                videoElement.srcObject = stream
                
                return new Promise((resolve) => {
                    videoElement.onloadeddata = () => {
                        resolve()
                    }
                })
            } catch (error) {
                console.error('Failed to start webcam for recalibration:', error)
                throw new Error('Unable to access camera for recalibration')
            }
        },

        drawPoint(x, y, radius) {
            const canvas = document.getElementById('canvas')
            const ctx = canvas.getContext('2d')
            
            // Don't clear the entire canvas, just draw the point
            ctx.fillStyle = this.$store.state.calibration.backgroundColor
            
            // Draw outer circle
            ctx.beginPath()
            ctx.strokeStyle = this.$store.state.calibration.pointColor
            ctx.fillStyle = this.$store.state.calibration.pointColor
            ctx.arc(x, y, radius, 0, Math.PI * 2, false)
            ctx.stroke()
            ctx.fill()
            
            // Draw inner red circle
            ctx.beginPath()
            ctx.strokeStyle = "red"
            ctx.fillStyle = "red"
            ctx.arc(x, y, 5, 0, Math.PI * 2, false)
            ctx.stroke()
            ctx.fill()
            
            // Draw hollow circumference
            ctx.strokeStyle = this.$store.state.calibration.pointColor
            ctx.lineWidth = 1
            ctx.beginPath()
            ctx.arc(x, y, this.$store.state.calibration.radius, 0, 2 * Math.PI, false)
            ctx.stroke()
        }
    },

    beforeDestroy() {
        // Clean up event listeners and overlays
        const canvas = document.getElementById('canvas')
        if (canvas) {
            canvas.removeEventListener('click', this.handleCanvasClick)
        }
        
        // Remove any existing overlays
        const overlays = document.querySelectorAll('[id*="recalibration"]')
        overlays.forEach(overlay => {
            if (document.body.contains(overlay)) {
                document.body.removeChild(overlay)
            }
        })
    }
}
</script>

<style>
.scroll-container {
    width: 100%;
    position: relative;
}

.instructions-overlay {
    position: fixed;
    top: 90px;
    right: 24px;
    z-index: 999;
    max-width: 300px;
}

.button-overlay {
    position: fixed;
    bottom: 24px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 12px;
}

.recalibration-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
}

.recalibration-instructions {
    background: white;
    padding: 30px;
    border-radius: 15px;
    text-align: center;
    max-width: 400px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.recalibration-instructions h3 {
    color: #FF425A;
    margin-bottom: 16px;
}

.recalibration-instructions p {
    margin-bottom: 12px;
    color: #333;
}

.recalibration-buttons {
    margin-top: 20px;
}
</style>