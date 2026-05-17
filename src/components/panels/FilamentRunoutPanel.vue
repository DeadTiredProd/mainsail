<template>
    <panel title="Filament Runout" icon="mdi-ray-vertex">
        
        <v-row>
            <v-col cols="12">
                <v-chip :color="connected ? 'success' : 'error'" dark>
                    {{ connected ? 'ESP32 Online' : 'ESP32 Offline' }}
                </v-chip>
            </v-col>

            <v-col cols="12">
                <v-chip :color="filament ? 'success' : 'error'" dark>
                    {{ filament ? 'Filament Present' : 'Filament Runout' }}
                </v-chip>
            </v-col>
        </v-row>

        <v-divider class="my-2" />

        <div>
            <div>Light: {{ light }}</div>
            <div>Threshold: {{ threshold }}</div>
        </div>

        <v-progress-linear
            class="mt-2"
            :value="progress"
            height="6"
        />
    </panel>
</template>

<script lang="ts">
import Component from 'vue-class-component'
import { Mixins } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'

@Component
export default class FilamentRunoutPanel extends Mixins(BaseMixin) {

    get connected() {
        return this.$store.getters['filament/isConnected']
    }

    get filament() {
        return this.$store.getters['filament/hasFilament']
    }

    get light() {
        return this.$store.getters['filament/light']
    }

    get threshold() {
        return this.$store.getters['filament/threshold']
    }

    get progress() {
        if (!this.threshold) return 0
        return Math.min((this.light / this.threshold) * 100, 100)
    }
}
</script>