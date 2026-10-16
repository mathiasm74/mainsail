<template>
    <div ref="root" class="map-control">
        <div class="d-flex justify-center" :style="{ gap: `${GAP}px` }">
            <!-- XY BED MAP -->
            <svg
                ref="xyMap"
                class="map-svg"
                :class="{ 'map-svg--active': xyActive }"
                :width="mapWidth"
                :height="mapHeight"
                :viewBox="`0 0 ${mapWidth} ${mapHeight}`"
                @pointerdown="onXyDown"
                @pointermove="onXyMove"
                @pointerup="onXyUp"
                @pointercancel="onXyCancel">
                <rect x="0" y="0" :width="mapWidth" :height="mapHeight" :fill="bgFill" />
                <g :stroke="fgColorFaint" stroke-width="1">
                    <line
                        v-for="i in GRID_CELLS - 1"
                        :key="`gx-${i}`"
                        :x1="(i / GRID_CELLS) * mapWidth"
                        :x2="(i / GRID_CELLS) * mapWidth"
                        y1="0"
                        :y2="mapHeight" />
                    <line
                        v-for="i in GRID_CELLS - 1"
                        :key="`gy-${i}`"
                        x1="0"
                        :x2="mapWidth"
                        :y1="(i / GRID_CELLS) * mapHeight"
                        :y2="(i / GRID_CELLS) * mapHeight" />
                </g>
                <rect
                    x="0.5"
                    y="0.5"
                    :width="mapWidth - 1"
                    :height="mapHeight - 1"
                    fill="none"
                    :stroke="fgColorLow"
                    stroke-width="1" />
                <circle
                    v-for="(point, index) in xySnapPoints"
                    :key="`snap-${index}`"
                    :cx="clampPx(xToPx(point[0]), mapWidth)"
                    :cy="clampPx(yToPx(point[1]), mapHeight)"
                    r="4"
                    :fill="primaryColor"
                    fill-opacity="0.5" />
                <template v-if="xyActive">
                    <!-- current toolhead position -->
                    <g :stroke="fgColorLow" stroke-width="1">
                        <line :x1="currentXPx" :x2="currentXPx" y1="0" :y2="mapHeight" />
                        <line x1="0" :x2="mapWidth" :y1="currentYPx" :y2="currentYPx" />
                    </g>
                    <circle :cx="currentXPx" :cy="currentYPx" r="7" :fill="currentColor" />
                    <!-- pending target while the head is still travelling -->
                    <g v-if="xyTarget && !xyDrag" :stroke="primaryColor" stroke-width="2" :fill="primaryColor">
                        <circle :cx="xyTargetPx[0]" :cy="xyTargetPx[1]" r="10" fill="none" />
                        <circle :cx="xyTargetPx[0]" :cy="xyTargetPx[1]" r="2" />
                    </g>
                    <!-- drag marker with live value -->
                    <g v-if="xyDrag" :stroke="primaryColor" stroke-width="2" :fill="primaryColor">
                        <circle :cx="xyDragPx[0]" :cy="xyDragPx[1]" r="10" fill="none" />
                        <circle :cx="xyDragPx[0]" :cy="xyDragPx[1]" r="2" />
                        <template v-if="dragLabelVisible">
                            <rect
                                :x="xyDragLabelBox[0]"
                                :y="xyDragLabelBox[1]"
                                :width="xyDragLabelBox[2]"
                                :height="LABEL_HEIGHT"
                                rx="3"
                                :fill="bgFill"
                                fill-opacity="0.9"
                                stroke="none" />
                            <text
                                :x="xyDragLabelPos[0]"
                                :y="xyDragLabelPos[1]"
                                :text-anchor="xyDragLabelPos[2]"
                                :fill="fgColorHi"
                                stroke="none"
                                class="map-label">
                                {{ xyDragLabel }}
                            </text>
                        </template>
                    </g>
                </template>
                <template v-else>
                    <rect x="0" y="0" :width="mapWidth" :height="mapHeight" fill="#000" fill-opacity="0.45" />
                    <text
                        :x="mapWidth / 2"
                        :y="mapHeight / 2"
                        text-anchor="middle"
                        dominant-baseline="middle"
                        :fill="fgColorHi"
                        class="map-label map-label--big">
                        {{ $t('Panels.ToolheadControlPanel.HomeFirst') }}
                    </text>
                </template>
            </svg>
            <!-- Z HEIGHT STRIP -->
            <svg
                ref="zMap"
                class="map-svg"
                :class="{ 'map-svg--active': zActive }"
                :width="Z_STRIP_WIDTH"
                :height="mapHeight"
                :viewBox="`0 0 ${Z_STRIP_WIDTH} ${mapHeight}`"
                @pointerdown="onZDown"
                @pointermove="onZMove"
                @pointerup="onZUp"
                @pointercancel="onZCancel">
                <rect x="0" y="0" :width="Z_STRIP_WIDTH" :height="mapHeight" :fill="bgFill" />
                <rect
                    x="0.5"
                    y="0.5"
                    :width="Z_STRIP_WIDTH - 1"
                    :height="mapHeight - 1"
                    fill="none"
                    :stroke="fgColorLow"
                    stroke-width="1" />
                <g v-for="line in zLines" :key="`zl-${line.z}`">
                    <line
                        x1="6"
                        :x2="Z_STRIP_WIDTH / 2 - line.gap"
                        :y1="line.y"
                        :y2="line.y"
                        :stroke="fgColorLow"
                        stroke-width="1" />
                    <line
                        :x1="Z_STRIP_WIDTH / 2 + line.gap"
                        :x2="Z_STRIP_WIDTH - 6"
                        :y1="line.y"
                        :y2="line.y"
                        :stroke="fgColorLow"
                        stroke-width="1" />
                    <text
                        :x="Z_STRIP_WIDTH / 2"
                        :y="line.y"
                        text-anchor="middle"
                        dominant-baseline="middle"
                        :fill="fgColorMid"
                        class="map-label map-label--small">
                        {{ line.label }}
                    </text>
                </g>
                <template v-if="zActive">
                    <g :stroke="currentColor" stroke-width="2" :fill="currentColor">
                        <line x1="4" :x2="Z_STRIP_WIDTH - 4" :y1="currentZPx" :y2="currentZPx" />
                        <circle :cx="Z_STRIP_WIDTH - 10" :cy="currentZPx" r="5" />
                    </g>
                    <line
                        v-if="zTarget !== null && !zDrag"
                        x1="4"
                        :x2="Z_STRIP_WIDTH - 4"
                        :y1="zToPx(zTarget)"
                        :y2="zToPx(zTarget)"
                        :stroke="primaryColor"
                        stroke-width="2" />
                    <g v-if="zDrag" :stroke="primaryColor" stroke-width="2">
                        <line x1="4" :x2="Z_STRIP_WIDTH - 4" :y1="zDragPx" :y2="zDragPx" />
                        <template v-if="dragLabelVisible">
                            <rect
                                :x="Z_STRIP_WIDTH / 2 - zDragLabelWidth / 2"
                                :y="zDragLabelY - LABEL_HEIGHT + 4"
                                :width="zDragLabelWidth"
                                :height="LABEL_HEIGHT"
                                rx="3"
                                :fill="bgFill"
                                fill-opacity="0.9"
                                stroke="none" />
                            <text
                                :x="Z_STRIP_WIDTH / 2"
                                :y="zDragLabelY"
                                text-anchor="middle"
                                :fill="fgColorHi"
                                stroke="none"
                                class="map-label">
                                {{ zDragLabel }}
                            </text>
                        </template>
                    </g>
                </template>
                <rect v-else x="0" y="0" :width="Z_STRIP_WIDTH" :height="mapHeight" fill="#000" fill-opacity="0.45" />
            </svg>
        </div>
        <div v-if="SAFE_Z > 0" class="text-right text--secondary caption mt-1 px-1">
            {{ $t('Panels.ToolheadControlPanel.TapsTravelAt', { z: SAFE_Z }) }}
        </div>
        <confirmation-dialog
            v-model="showHeadroomDialog"
            :icon="mdiArrowUpBold"
            :title="$t('Panels.ToolheadControlPanel.HeadroomTitle')"
            :text="$t('Panels.ToolheadControlPanel.HeadroomText', { z: pendingZLabel, max: zMax.toFixed(0) })"
            :action-button-text="$t('Buttons.Yes')"
            :cancel-button-text="$t('Buttons.No')"
            action-button-color="primary"
            @action="confirmHeadroom" />
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Ref, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import ConfirmationDialog from '@/components/dialogs/ConfirmationDialog.vue'
import ControlMixin from '@/components/mixins/control'
import ThemeMixin from '@/components/mixins/theme'
import { defaultPrimaryColor } from '@/store/variables'
import { mdiArrowUpBold } from '@mdi/js'

type Point = [number, number]

interface DragState {
    x0: number
    y0: number
    x: number
    y: number
}

interface ZLine {
    z: number
    y: number
    label: string
    gap: number
}

// taps on the bed map lift the nozzle to at least this height before travelling
const SAFE_Z = 15
const Z_STRIP_WIDTH = 64
const Z_PAD = 14
const GAP = 12
const GRID_CELLS = 6
const MAX_MAP_HEIGHT = 270
const MIN_MAP_HEIGHT = 160
// keeps rings drawn at the bed edge fully inside the map
const EDGE_INSET_PX = 10
// approximate glyph width of the 15px label font, used to size label backgrounds
const LABEL_CHAR_PX = 8.5
const LABEL_HEIGHT = 20
// pointer travel below this is a tap (snaps to reference points), above it a drag
const TAP_TOLERANCE_PX = 8
const SNAP_RADIUS_PX = 15
const Z_SNAP_PX = 14
const Z_LINE_MIN_SPACING_PX = 22
const Z_LINE_CANDIDATES = [0, 10, 25, 50, 100, 150, 200, 250, 300, 350, 400, 500, 600, 800, 1000]
// the drag label appears only once the touch has lived this long, so quick taps don't flash it
const DRAG_LABEL_DELAY_MS = 250
// pending target markers vanish once the head is this close
const XY_ARRIVED_MM = 0.8
const Z_ARRIVED_MM = 0.3
// Z targets above this fraction of the Z range ask for confirmation that there is headroom
const Z_CONFIRM_FRACTION = 1 / 3

@Component({
    components: { ConfirmationDialog },
})
export default class MapControl extends Mixins(BaseMixin, ControlMixin, ThemeMixin) {
    @Ref() readonly root!: HTMLDivElement
    @Ref() readonly xyMap!: SVGSVGElement
    @Ref() readonly zMap!: SVGSVGElement

    mdiArrowUpBold = mdiArrowUpBold

    SAFE_Z = SAFE_Z
    Z_STRIP_WIDTH = Z_STRIP_WIDTH
    GAP = GAP
    GRID_CELLS = GRID_CELLS
    LABEL_HEIGHT = LABEL_HEIGHT

    containerWidth = 0
    xyDrag: DragState | null = null
    zDrag: DragState | null = null
    xyTarget: Point | null = null
    zTarget: number | null = null
    dragLabelVisible = false
    showHeadroomDialog = false
    pendingZ: number | null = null
    dragLabelTimer: number | null = null
    resizeObserver: ResizeObserver | null = null

    get primaryColor(): string {
        return this.$store.state.gui.theme?.primary ?? defaultPrimaryColor
    }

    get currentColor(): string {
        return '#4caf50'
    }

    get bgFill(): string {
        return this.$vuetify.theme.dark ? '#10141b' : '#f2f4f7'
    }

    get isPrinting(): boolean {
        return ['printing'].includes(this.printer_state)
    }

    get xyActive(): boolean {
        return this.xAxisHomed && this.yAxisHomed
    }

    get zActive(): boolean {
        return this.zAxisHomed
    }

    get axisMinimum(): number[] {
        return this.$store.state.printer.toolhead?.axis_minimum ?? []
    }

    get axisMaximum(): number[] {
        return this.$store.state.printer.toolhead?.axis_maximum ?? []
    }

    /**
     * The map represents the bed, not the travel area: axis limits often exceed the
     * physical plate, which would put the center snap point off the bed's center.
     * A configured bed size overrides the axis limits.
     */
    get mapBedSizeX(): number | null {
        return this.$store.state.gui.control.mapBedSizeX ?? null
    }

    get mapBedSizeY(): number | null {
        return this.$store.state.gui.control.mapBedSizeY ?? null
    }

    get bedXMin(): number {
        return this.mapBedSizeX ? 0 : (this.axisMinimum[0] ?? 0)
    }

    get bedXMax(): number {
        return this.mapBedSizeX ?? this.axisMaximum[0] ?? 200
    }

    get bedYMin(): number {
        return this.mapBedSizeY ? 0 : (this.axisMinimum[1] ?? 0)
    }

    get bedYMax(): number {
        return this.mapBedSizeY ?? this.axisMaximum[1] ?? 200
    }

    get zMin(): number {
        return Math.max(this.axisMinimum[2] ?? 0, 0)
    }

    get zMax(): number {
        return this.axisMaximum[2] ?? 200
    }

    get bedXSpan(): number {
        return Math.max(this.bedXMax - this.bedXMin, 1)
    }

    get bedYSpan(): number {
        return Math.max(this.bedYMax - this.bedYMin, 1)
    }

    get mapWidth(): number {
        const available = Math.max(this.containerWidth - Z_STRIP_WIDTH - GAP, 0)
        const widthForMaxHeight = (MAX_MAP_HEIGHT * this.bedXSpan) / this.bedYSpan

        return Math.max(Math.floor(Math.min(available, widthForMaxHeight)), 1)
    }

    get mapHeight(): number {
        const height = (this.mapWidth * this.bedYSpan) / this.bedXSpan

        return Math.max(Math.floor(Math.min(height, MAX_MAP_HEIGHT)), MIN_MAP_HEIGHT)
    }

    get xySnapPoints(): Point[] {
        return [
            [this.bedXMin, this.bedYMin],
            [this.bedXMax, this.bedYMin],
            [this.bedXMin, this.bedYMax],
            [this.bedXMax, this.bedYMax],
            [(this.bedXMin + this.bedXMax) / 2, (this.bedYMin + this.bedYMax) / 2],
        ]
    }

    /**
     * live_position follows the physical toolhead during moves, while gcode_position jumps
     * to the target immediately. The homing origin is subtracted so values stay in gcode coordinates.
     */
    get position(): number[] {
        const live = this.$store.state.printer.motion_report?.live_position ?? null
        const origin = this.$store.state.printer.gcode_move?.homing_origin ?? null
        const gcode = this.$store.state.printer.gcode_move?.gcode_position ?? [0, 0, 0]

        const pos: number[] = live && origin ? live.map((v: number, i: number) => v - (origin[i] ?? 0)) : gcode

        return pos.map((v) => (Math.abs(v) < 0.005 ? 0 : v))
    }

    get currentXPx(): number {
        return this.xToPx(Math.min(Math.max(this.position[0], this.bedXMin), this.bedXMax))
    }

    get currentYPx(): number {
        return this.yToPx(Math.min(Math.max(this.position[1], this.bedYMin), this.bedYMax))
    }

    get currentZPx(): number {
        return this.zToPx(this.position[2])
    }

    get xyTargetPx(): Point {
        if (!this.xyTarget) return [0, 0]

        return [
            this.insetPx(this.xToPx(this.xyTarget[0]), this.mapWidth),
            this.insetPx(this.yToPx(this.xyTarget[1]), this.mapHeight),
        ]
    }

    get xyDragPx(): Point {
        if (!this.xyDrag) return [0, 0]

        return [this.insetPx(this.xyDrag.x, this.mapWidth), this.insetPx(this.xyDrag.y, this.mapHeight)]
    }

    get xyDragLabel(): string {
        if (!this.xyDrag) return ''
        const [x, y] = this.pxToBed(this.xyDrag.x, this.xyDrag.y, false)

        return `X ${x.toFixed(0)} · Y ${y.toFixed(0)}`
    }

    get xyDragLabelWidth(): number {
        return this.xyDragLabel.length * LABEL_CHAR_PX + 10
    }

    get xyDragLabelPos(): [number, number, string] {
        const [px, py] = this.xyDragPx
        const flipX = px > this.mapWidth - this.xyDragLabelWidth - 24
        const x = flipX ? px - 20 : px + 20
        const y = py > 44 ? py - 22 : py + 36

        return [x, y, flipX ? 'end' : 'start']
    }

    get xyDragLabelBox(): [number, number, number] {
        const [x, y, anchor] = this.xyDragLabelPos
        const width = this.xyDragLabelWidth
        const left = anchor === 'end' ? x - width + 5 : x - 5

        return [left, y - LABEL_HEIGHT + 4, width]
    }

    get zDragPx(): number {
        if (!this.zDrag) return 0

        return Math.min(Math.max(this.zDrag.y, Z_PAD), this.mapHeight - Z_PAD)
    }

    get zDragLabel(): string {
        return this.pxToZ(this.zDragPx).toFixed(1)
    }

    get zDragLabelWidth(): number {
        return this.zDragLabel.length * LABEL_CHAR_PX + 10
    }

    get zDragLabelY(): number {
        return this.zDragPx > 32 ? this.zDragPx - 8 : this.zDragPx + 22
    }

    get pendingZLabel(): string {
        return this.pendingZ?.toFixed(1) ?? ''
    }

    get zLines(): ZLine[] {
        const values = Z_LINE_CANDIDATES.filter((z) => z >= this.zMin && z < this.zMax)
        values.push(this.zMax)

        const lines: ZLine[] = []
        let lastY = Number.POSITIVE_INFINITY
        for (const z of values) {
            const y = this.zToPx(z)
            // drop reference lines that would collide with their neighbour on short strips
            if (lastY - y < Z_LINE_MIN_SPACING_PX && z !== this.zMax) continue
            if (lastY - y < Z_LINE_MIN_SPACING_PX) lines.pop()

            const label = z.toFixed(0)
            lines.push({ z, y, label, gap: label.length * 4 + 8 })
            lastY = y
        }

        return lines
    }

    @Watch('position')
    onPositionChange(pos: number[]): void {
        if (
            this.xyTarget &&
            Math.abs(pos[0] - this.xyTarget[0]) < XY_ARRIVED_MM &&
            Math.abs(pos[1] - this.xyTarget[1]) < XY_ARRIVED_MM
        ) {
            this.xyTarget = null
        }
        if (this.zTarget !== null && Math.abs(pos[2] - this.zTarget) < Z_ARRIVED_MM) {
            this.zTarget = null
        }
    }

    @Watch('showHeadroomDialog')
    onHeadroomDialogChange(visible: boolean): void {
        // closing without confirming drops the pending target
        if (!visible) this.pendingZ = null
    }

    @Watch('homedAxes')
    onHomedChange(): void {
        this.xyTarget = null
        this.zTarget = null
    }

    mounted(): void {
        this.resizeObserver = new ResizeObserver(this.onResize)
        this.resizeObserver.observe(this.root)
        this.containerWidth = this.root.clientWidth
    }

    beforeDestroy(): void {
        this.resizeObserver?.disconnect()
        this.resizeObserver = null
        this.clearDragLabelTimer()
    }

    onResize(entries: ResizeObserverEntry[]): void {
        const width = entries[0]?.contentRect.width ?? 0
        if (width > 0) this.containerWidth = width
    }

    /**
     * Coordinate conversions
     */
    xToPx(x: number): number {
        return ((x - this.bedXMin) / this.bedXSpan) * this.mapWidth
    }

    yToPx(y: number): number {
        return this.mapHeight - ((y - this.bedYMin) / this.bedYSpan) * this.mapHeight
    }

    clampPx(value: number, max: number): number {
        return Math.min(Math.max(value, 0), max)
    }

    insetPx(value: number, max: number): number {
        return Math.min(Math.max(value, EDGE_INSET_PX), max - EDGE_INSET_PX)
    }

    pxToBed(px: number, py: number, snap: boolean): Point {
        const x = this.bedXMin + (px / this.mapWidth) * this.bedXSpan
        const y = this.bedYMin + (1 - py / this.mapHeight) * this.bedYSpan

        if (snap) {
            const snapX = (SNAP_RADIUS_PX / this.mapWidth) * this.bedXSpan
            const snapY = (SNAP_RADIUS_PX / this.mapHeight) * this.bedYSpan
            const hit = this.xySnapPoints.find((p) => Math.abs(x - p[0]) < snapX && Math.abs(y - p[1]) < snapY)
            if (hit) return hit
        }

        return [Math.min(Math.max(x, this.bedXMin), this.bedXMax), Math.min(Math.max(y, this.bedYMin), this.bedYMax)]
    }

    /**
     * The Z strip uses a square-root scale: the low end, where most manual
     * moves happen, gets more room than the top of a tall Z axis.
     */
    zToPx(z: number): number {
        const span = Math.max(this.mapHeight - 2 * Z_PAD, 1)
        const frac = Math.sqrt(Math.min(Math.max((z - this.zMin) / (this.zMax - this.zMin || 1), 0), 1))

        return this.mapHeight - Z_PAD - frac * span
    }

    pxToZ(py: number): number {
        const span = Math.max(this.mapHeight - 2 * Z_PAD, 1)
        const frac = Math.min(Math.max((this.mapHeight - Z_PAD - py) / span, 0), 1)

        return this.zMin + frac * frac * (this.zMax - this.zMin)
    }

    localPoint(svg: SVGSVGElement, event: PointerEvent): Point {
        const rect = svg.getBoundingClientRect()

        return [event.clientX - rect.left, event.clientY - rect.top]
    }

    /**
     * Drag label timing
     */
    startDragLabelTimer(): void {
        this.clearDragLabelTimer()
        this.dragLabelVisible = false
        this.dragLabelTimer = window.setTimeout(() => {
            this.dragLabelVisible = true
        }, DRAG_LABEL_DELAY_MS)
    }

    clearDragLabelTimer(): void {
        if (this.dragLabelTimer !== null) window.clearTimeout(this.dragLabelTimer)
        this.dragLabelTimer = null
        this.dragLabelVisible = false
    }

    /**
     * XY map interaction
     */
    onXyDown(event: PointerEvent): void {
        if (!this.xyActive || this.isPrinting) return

        const [x, y] = this.localPoint(this.xyMap, event)
        this.xyDrag = { x0: x, y0: y, x, y }
        this.xyMap.setPointerCapture(event.pointerId)
        this.startDragLabelTimer()
        event.preventDefault()
    }

    onXyMove(event: PointerEvent): void {
        if (!this.xyDrag) return

        const [x, y] = this.localPoint(this.xyMap, event)
        this.xyDrag.x = x
        this.xyDrag.y = y
    }

    onXyUp(event: PointerEvent): void {
        if (!this.xyDrag) return

        const { x0, y0 } = this.xyDrag
        const [x, y] = this.localPoint(this.xyMap, event)
        this.xyDrag = null
        this.clearDragLabelTimer()

        const tapped = Math.abs(x - x0) <= TAP_TOLERANCE_PX && Math.abs(y - y0) <= TAP_TOLERANCE_PX
        const target = this.pxToBed(x, y, tapped)
        this.moveXY(target)
    }

    onXyCancel(): void {
        this.xyDrag = null
        this.clearDragLabelTimer()
    }

    moveXY(target: Point): void {
        const moves: { axes: string; feedrate: number }[] = []
        if (SAFE_Z > 0 && this.zAxisHomed && this.position[2] < SAFE_Z) {
            moves.push({ axes: `Z${SAFE_Z}`, feedrate: this.feedrateZ })
        }
        moves.push({ axes: `X${target[0].toFixed(1)} Y${target[1].toFixed(1)}`, feedrate: this.feedrateXY })

        this.sendAbsoluteMoves(moves)
        this.xyTarget = target
    }

    /**
     * Z strip interaction
     */
    onZDown(event: PointerEvent): void {
        if (!this.zActive || this.isPrinting) return

        const [x, y] = this.localPoint(this.zMap, event)
        this.zDrag = { x0: x, y0: y, x, y }
        this.zMap.setPointerCapture(event.pointerId)
        this.startDragLabelTimer()
        event.preventDefault()
    }

    onZMove(event: PointerEvent): void {
        if (!this.zDrag) return

        const [x, y] = this.localPoint(this.zMap, event)
        this.zDrag.x = x
        this.zDrag.y = y
    }

    onZUp(event: PointerEvent): void {
        if (!this.zDrag) return

        const y0 = this.zDrag.y0
        const [, y] = this.localPoint(this.zMap, event)
        this.zDrag = null
        this.clearDragLabelTimer()

        let z = this.pxToZ(Math.min(Math.max(y, Z_PAD), this.mapHeight - Z_PAD))
        if (Math.abs(y - y0) <= TAP_TOLERANCE_PX) {
            // a tap snaps to the nearest reference line when close to one
            const hit = this.zLines.find((line) => Math.abs(line.y - y) <= Z_SNAP_PX)
            if (hit) z = hit.z
        }
        z = Math.min(Math.max(z, this.zMin), this.zMax)

        if (z > this.zMax * Z_CONFIRM_FRACTION) {
            this.pendingZ = z
            this.showHeadroomDialog = true
            return
        }

        this.moveZ(z)
    }

    confirmHeadroom(): void {
        if (this.pendingZ === null) return

        this.moveZ(this.pendingZ)
        this.pendingZ = null
    }

    moveZ(z: number): void {
        this.sendAbsoluteMoves([{ axes: `Z${z.toFixed(1)}`, feedrate: this.feedrateZ }])
        this.zTarget = z
    }

    onZCancel(): void {
        this.zDrag = null
        this.clearDragLabelTimer()
    }

    /**
     * Sends absolute moves in sequence, mirroring the coordinate fields' gcode
     */
    sendAbsoluteMoves(moves: { axes: string; feedrate: number }[]): void {
        const gcode: string[] = []

        if (this.existsClientLinearMoveMacro) {
            for (const move of moves) {
                const params = move.axes
                    .split(' ')
                    .map((part) => `${part.slice(0, 1)}=${part.slice(1)}`)
                    .join(' ')
                gcode.push(`_CLIENT_LINEAR_MOVE ${params} F=${move.feedrate * 60} ABSOLUTE=1`)
            }
        } else {
            gcode.push('SAVE_GCODE_STATE NAME=_ui_movement')
            gcode.push('G90')
            for (const move of moves) gcode.push(`G1 ${move.axes} F${move.feedrate * 60}`)
            gcode.push('RESTORE_GCODE_STATE NAME=_ui_movement')
        }

        this.doSend(gcode.join('\n'))
    }
}
</script>

<style scoped>
.map-svg {
    display: block;
    flex: 0 0 auto;
    border-radius: 4px;
    user-select: none;
    touch-action: none;
}

.map-svg--active {
    cursor: crosshair;
}

.map-label {
    font-size: 15px;
    font-weight: 500;
    pointer-events: none;
}

.map-label--big {
    font-size: 22px;
}

.map-label--small {
    font-size: 11px;
    font-weight: 400;
}
</style>
