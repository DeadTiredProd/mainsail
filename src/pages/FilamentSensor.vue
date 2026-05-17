<template>
    <div class="d-flex flex-column">

        <v-row>
            <v-col>
                <v-card>

                    <v-card-title class="d-flex align-center">
                        <v-icon class="mr-2">{{ mdiWrench }}</v-icon>
                        Filament Sensor (ESP32)
                    </v-card-title>

                    <v-divider />

                    <v-card-text>

                        <!-- STATUS -->
                        <v-row>
                            <v-col cols="12" md="4">
                                <v-chip :color="connected ? 'success' : 'error'" dark>
                                    {{ connected ? 'ESP32 Connected' : 'ESP32 Offline' }}
                                </v-chip>
                            </v-col>

                            <v-col cols="12" md="4">
                                <v-chip :color="override ? 'warning' : 'primary'" dark>
                                    {{ override ? 'Override ON' : 'Override OFF' }}
                                </v-chip>
                            </v-col>

                            <v-col cols="12" md="4">
                                <v-chip :color="filament ? 'success' : 'error'" dark>
                                    {{ filament ? 'Filament Present' : 'Filament Missing' }}
                                </v-chip>
                            </v-col>
                        </v-row>

                        <!-- SENSOR INFO -->
                        <v-row class="mt-4">
                            <v-col cols="12" md="6">
                                Light Level: <b>{{ light }}</b>
                            </v-col>

                            <v-col cols="12" md="6">
                                Threshold: <b>{{ threshold }}</b>
                            </v-col>
                        </v-row>

                        <v-row class="mt-2">
                            <v-col cols="12">
                                <v-slider
                                    v-model="threshold"
                                    min="0"
                                    max="4095"
                                    step="5"
                                    label="Threshold"
                                    @change="updateThreshold"
                                />
                            </v-col>
                        </v-row>

                        <v-row class="mt-2">
                            <v-col cols="12">
                                <v-text-field
                                    v-model="sensorIP"
                                    label="ESP32 IP Address"
                                    outlined
                                    dense
                                />
                            </v-col>
                        </v-row>

                        <v-row class="mt-2">
                            <v-col cols="12" md="6">
                                <v-btn block color="success" @click="enableOverride">
                                    Enable Override
                                </v-btn>
                            </v-col>

                            <v-col cols="12" md="6">
                                <v-btn block color="error" @click="disableOverride">
                                    Disable Override
                                </v-btn>
                            </v-col>
                        </v-row>

                        <v-row class="mt-2">
                            <v-col cols="12">
                                <v-btn block color="primary" @click="pollESP">
                                    Refresh Status
                                </v-btn>
                            </v-col>
                        </v-row>

                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>

        <!-- VISUAL -->
        <v-row>
            <v-col>
                <v-card class="mt-4 pa-4">
                    <v-card-title>Filament Sensor Cross-Section</v-card-title>

                    <div class="svg-wrap">
                        <svg viewBox="0 0 400 120" class="filament-svg">

                            <image
                                :href="require('@/assets/filamentsensor.svg')"
                                x="0"
                                y="0"
                                width="200"
                                height="120"
                            />

                            <rect
                                x="100"
                                y="59"
                                width="27"
                                height="2"
                                class="light-beam"
                                :class="{ blocked: filament }"
                            />

                            <g class="filament" :class="{ active: filament }">

                                <image
                                    :href="require('@/assets/filament.svg')"
                                    x="-16.7"
                                    y="45"
                                    width="220"
                                    height="10"
                                />

                                <rect
                                    x="93.7"
                                    y="45.1"
                                    width="0.3"
                                    height="10"
                                    fill="black"
                                    :style="{ opacity: filament ? 0.8 : 0 }"
                                />

                            </g>

                            <circle
                                cx="100"
                                cy="59.8"
                                r="4"
                                fill="cyan"
                                :class="{ dim: filament }"
                            />

                        </svg>
                    </div>

                    <div class="text-center mt-2">
                        {{ filament ? 'Filament Present' : 'Filament Missing' }}
                    </div>
                </v-card>
            </v-col>
        </v-row>

        <!-- RUNOUT DIALOG -->
        <FilamentRunoutDialog
            v-model="runoutDialog"
            @resume="resumePrint"
            @cancel="handleCancel"
        />

    </div>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import FilamentRunoutDialog from '@/components/dialogs/FilamentRunoutDialog.vue'
import { mdiWrench } from '@mdi/js'
import { filamentBus } from '@/bus/filamentBus'



@Component({
    components: {
        FilamentRunoutDialog
    }
})
export default class FilamentSensor extends Mixins(BaseMixin) {
    mdiWrench = mdiWrench

    sensorIP = '192.168.1.227'

    connected = false
    filament = false
    override = false

    light = 0
    threshold = 2800

    consoleData = ''
    rawStatus = ''

    runoutDialog = false
    wasFilamentPresent = true   // IMPORTANT: assume filament exists at boot

    mounted() {
        const saved = localStorage.getItem('esp32_ip')
        if (saved) this.sensorIP = saved

        this.pollESP()
        setInterval(() => this.pollESP(), 1500)
    }

    @Watch('sensorIP')
    onIPChange(val: string) {
        localStorage.setItem('esp32_ip', val)
    }

    async call(path: string) {
        try {
            const res = await fetch(`http://${this.sensorIP}${path}`)
            return await res.text()
        } catch {
            return null
        }
    }

    async enableOverride() {
        await this.call('/enable_override')
        this.override = true
    }

    async disableOverride() {
        await this.call('/disable_override')
        this.override = false
    }

    async updateThreshold() {
        await fetch(`http://${this.sensorIP}/set_threshold?value=${this.threshold}`)
    }

    async pollESP() {
        const status = await this.call('/status')

        if (!status) {
            this.connected = false
            return
        }

        this.rawStatus = status

        try {
            const data = JSON.parse(status)

            this.connected = data.wifi === true || data.wifi === "true"

            const newFilament =
                data.filament === true || data.filament === "true"

            this.light = Number(data.light ?? 0)
            this.threshold = Number(data.threshold ?? this.threshold)

            // ✅ RUNOUT DETECTION (FIXED EDGE LOGIC)
            if (this.wasFilamentPresent && !newFilament) {
				filamentBus.$emit('show-filament-dialog')
			}

            this.wasFilamentPresent = newFilament
            this.filament = newFilament

        } catch {
            this.connected = false
        }

        const consoleRes = await this.call('/console')
        if (consoleRes) this.consoleData = consoleRes
    }

    async resumePrint() {
        await fetch(`http://${this.sensorIP}/resume`)
        this.runoutDialog = false
    }

    handleCancel() {
        this.runoutDialog = false
    }
}
</script>

<style scoped>
.logbox {
    max-height: 300px;
    overflow: auto;
    background: black;
    color: lime;
    padding: 10px;
    font-size: 12px;
}

.filament {
    transform: translateX(-447px) translateY(450px) scale(6, 8);
    transition: transform 0.6s ease;
    will-change: transform;
}

.filament.active {
    transform: translateX(-447px) translateY(-339px) scale(6, 8);
}

.light-beam {
    fill: cyan;
    opacity: 1;
    transition: opacity 0.4s ease;
}

.light-beam.blocked {
    opacity: 0.2;
}

.dim {
    opacity: 0.3;
    transition: opacity 0.4s ease;
}
</style>