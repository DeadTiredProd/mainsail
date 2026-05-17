<template>
    <v-dialog v-model="showDialog" width="500" persistent :fullscreen="isMobile">
        <panel
            card-class="filament-change-dialog"
            :icon="mdiAlert"
            title="Filament Change Required"
            :margin-bottom="false"
        >
            <template #buttons>
                <v-btn icon tile @click="close">
                    <v-icon>{{ mdiCloseThick }}</v-icon>
                </v-btn>
            </template>

            <v-card-text>

                <div class="text-center mb-3">
                    <h3>Filament Runout Detected</h3>
                </div>

                <!-- TIMER -->
                <div class="timer-box">
                    <div class="label">Time since runout:</div>
                    <div class="time">{{ formattedTime }}</div>

                    <v-progress-linear
                        :value="timerPercent"
                        height="8"
                        color="orange"
                        class="mt-2"
                    />
                </div>

                <!-- TEMPERATURES -->
                <v-row class="mt-4">
                    <v-col cols="6">
                        <v-card outlined class="pa-2 text-center">
                            <div>Nozzle</div>
                            <strong>{{ nozzleTemp }}°C</strong>
                        </v-card>
                    </v-col>

                    <v-col cols="6">
                        <v-card outlined class="pa-2 text-center">
                            <div>Bed</div>
                            <strong>{{ bedTemp }}°C</strong>
                        </v-card>
                    </v-col>
                </v-row>

                <!-- HEATER STATUS -->
                <v-alert
                    :type="heaterArmed ? 'success' : 'warning'"
                    dense
                    class="mt-4"
                >
                    {{ heaterArmed ? 'Printer is ready / heated' : 'Printer idle / cooling' }}
                </v-alert>

                <!-- ACTIONS -->
                <div class="actions mt-4">

                    <v-btn
                        block
                        color="primary"
                        :disabled="heaterArmed"
                        @click="preheat"
                    >
                        Preheat (210°C nozzle / 65°C bed)
                    </v-btn>

                    <v-btn
                        block
                        color="success"
                        class="mt-3"
                        @click="confirmChange"
                    >
                        Confirm Filament Loaded & Resume Print
                    </v-btn>

                    <v-btn
                        block
                        color="error"
                        class="mt-3"
                        @click="cancelAndCool"
                    >
                        Cancel & Cool Down
                    </v-btn>

                </div>

                <v-alert
                    v-if="autoShutdownWarning"
                    type="error"
                    class="mt-4"
                    dense
                >
                    No confirmation received. Heating has been disabled.
                </v-alert>

            </v-card-text>
        </panel>
    </v-dialog>
</template>

<script lang="ts">
import Component from 'vue-class-component'
import Panel from '@/components/ui/Panel.vue'
import { Mixins, VModel, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { mdiAlert, mdiCloseThick } from '@mdi/js'
import Vue from 'vue'
@Component({
    components: { Panel },
})
export default class FilamentChangeDialog extends Mixins(BaseMixin) {
    mdiAlert = mdiAlert
    mdiCloseThick = mdiCloseThick

    @VModel({ type: Boolean }) showDialog!: boolean

    // ===== TIMER =====
    startTime = 0
    now = Date.now()
    timer: any = null

    // ===== TEMP REFRESH =====
    tempTimer: any = null

    maxTime = 120000

    // ===== STATE =====
    heaterArmed = false
    autoShutdownWarning = false

    printerIP = '192.168.1.33'

    nozzleTemp = 0
    bedTemp = 0

    // ===== OPEN / CLOSE =====
    @Watch('showDialog')
    onDialogChange(val: boolean) {
        if (val) {
            this.resetAndStart()
            this.startTempPolling()
        } else {
            this.stopAll()
        }
    }

    mounted() {
        if (this.showDialog) {
            this.resetAndStart()
            this.startTempPolling()
        }
    }

    beforeDestroy() {
        this.stopAll()
    }

    // ===== TIMER =====
    resetAndStart() {
        this.stopTimer()
        this.startTime = Date.now()
        this.now = Date.now()

        this.timer = setInterval(() => {
            this.now = Date.now()
        }, 250)
    }

    stopTimer() {
        if (this.timer) clearInterval(this.timer)
        this.timer = null
    }

    // ===== TEMP POLLING =====
    startTempPolling() {
        this.fetchTemps()

        this.tempTimer = setInterval(() => {
            this.fetchTemps()
        }, 2000)
    }

    stopTemps() {
        if (this.tempTimer) clearInterval(this.tempTimer)
        this.tempTimer = null
    }

    async fetchTemps() {
        try {
            const res = await fetch(
                `http://${this.printerIP}:7125/printer/objects/query?extruder&heater_bed`
            )

            const json = await res.json()

            this.nozzleTemp =
                json?.result?.status?.extruder?.temperature ?? 0

            this.bedTemp =
                json?.result?.status?.heater_bed?.temperature ?? 0

        } catch (e) {
            console.error('Temp fetch failed', e)
        }
    }

    // ===== DERIVED =====
    get elapsed() {
        return this.now - this.startTime
    }

    get formattedTime() {
        const sec = Math.floor(this.elapsed / 1000)
        const m = Math.floor(sec / 60)
        const s = sec % 60
        return `${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`
    }

    get timerPercent() {
        return Math.min((this.elapsed / this.maxTime) * 100, 100)
    }

    // ===== ACTIONS =====
    async preheat() {
        await fetch(`http://${this.printerIP}:7125/printer/gcode/script?script=M104 S210`)
        await fetch(`http://${this.printerIP}:7125/printer/gcode/script?script=M140 S65`)

        this.heaterArmed = true
    }

    async confirmChange() {
        await fetch(`http://${this.printerIP}:7125/printer/gcode/script?script=RESUME`)

        this.close()
    }

    async cancelAndCool() {
        await fetch(`http://${this.printerIP}:7125/printer/gcode/script?script=TURN_OFF_HEATERS`)

        this.heaterArmed = false
    }

    // ===== CLEANUP =====
    stopAll() {
        this.stopTimer()
        this.stopTemps()
    }

    close() {
        this.showDialog = false
        this.stopAll()
    }
}
</script>

<style scoped>
.timer-box {
    text-align: center;
    padding: 10px;
    background: #111;
    border-radius: 8px;
    color: white;
}

.time {
    font-size: 28px;
    font-weight: bold;
    margin-top: 5px;
}

.actions {
    margin-top: 10px;
}
</style>