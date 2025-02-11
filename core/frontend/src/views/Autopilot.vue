<template>
  <v-card
    elevation="0"
    class="mx-auto my-12 px-1 py-6 text-center"
    min-width="370"
    max-width="720"
  >
    <v-card>
      <v-card-title>Autopilot</v-card-title>
      <v-card-text>
        <span v-if="board_undefined">No board running</span>
        <div v-else>
          <v-row
            v-for="(value, name) in autopilot_info"
            :key="name"
            class="ma-0"
            no-gutters
          >
            <v-col class="text-right ma-0 font-weight-bold">
              {{ name }}:
            </v-col>
            <v-col class="text-left ml-1">
              {{ value }}
            </v-col>
          </v-row>
          <br>
        </div>
        <v-expansion-panels>
          <v-expansion-panel>
            <v-expansion-panel-header>
              Firmware update
            </v-expansion-panel-header>
            <v-expansion-panel-content>
              <firmware-manager />
            </v-expansion-panel-content>
          </v-expansion-panel>
        </v-expansion-panels>
        <v-expansion-panels v-if="settings.is_pirate_mode && isLinuxFlightController">
          <v-expansion-panel>
            <v-expansion-panel-header>
              Serial Port Configuration
            </v-expansion-panel-header>
            <v-expansion-panel-content>
              <autopilot-serial-configuration />
            </v-expansion-panel-content>
          </v-expansion-panel>
        </v-expansion-panels>
      </v-card-text>
      <v-card-actions class="d-flex justify-end align-center flex-wrap">
        <v-spacer />
        <v-btn
          v-if="settings.is_pirate_mode"
          color="secondary"
          :block="$vuetify.breakpoint.xs"
          class="ma-1"
          :disabled="restarting"
          @click="openBoardChangeDialog"
        >
          Change board
        </v-btn>
        <v-btn
          v-if="settings.is_pirate_mode"
          class="ma-1"
          :block="$vuetify.breakpoint.xs"
          color="secondary"
          :disabled="restarting"
          @click="start_autopilot"
        >
          Start autopilot
        </v-btn>
        <v-btn
          v-if="settings.is_pirate_mode"
          class="ma-1"
          :block="$vuetify.breakpoint.xs"
          color="secondary"
          :disabled="restarting"
          @click="stop_autopilot"
        >
          Stop autopilot
        </v-btn>
        <v-btn
          color="primary"
          class="ma-1"
          :block="$vuetify.breakpoint.xs"
          :disabled="restarting"
          @click="restart_autopilot"
        >
          Restart autopilot
        </v-btn>
      </v-card-actions>
    </v-card>
    <board-change-dialog
      v-model="show_board_change_dialog"
    />
  </v-card>
</template>

<script lang="ts">
import Vue from 'vue'

import {
  fetchAvailableBoards, fetchCurrentBoard, fetchFirmwareInfo, fetchVehicleType,
} from '@/components/autopilot/AutopilotManagerUpdater'
import AutopilotSerialConfiguration from '@/components/autopilot/AutopilotSerialConfiguration.vue'
import BoardChangeDialog from '@/components/autopilot/BoardChangeDialog.vue'
import FirmwareManager from '@/components/autopilot/FirmwareManager.vue'
import Notifier from '@/libs/notifier'
import settings from '@/libs/settings'
import autopilot_data from '@/store/autopilot'
import autopilot from '@/store/autopilot_manager'
import { FirmwareInfo, FlightController } from '@/types/autopilot'
import { autopilot_service } from '@/types/frontend_services'
import back_axios from '@/utils/api'
import { callPeriodically, stopCallingPeriodically } from '@/utils/helper_functions'

const notifier = new Notifier(autopilot_service)

export default Vue.extend({
  name: 'Autopilot',
  components: {
    BoardChangeDialog,
    FirmwareManager,
    AutopilotSerialConfiguration,
  },
  data() {
    return {
      settings,
      show_board_change_dialog: false,
    }
  },
  computed: {
    autopilot_info(): Record<string, string> {
      let version = 'Unknown'
      if (this.firmware_info) {
        version = `${this.firmware_info.version} (${this.firmware_info.type})`
      }

      const record: Record<string, string> = {
        'Board name': this.current_board?.name ?? 'Unknown',
        Manufacturer: this.current_board?.manufacturer ?? 'Unknown',
        'Mavlink platform': this.current_board?.platform ?? 'Unknown',
        'Firmware version': version,
        'Vehicle type': this.vehicle_type ?? 'Unknown',
      }

      if (this.current_board?.path) {
        record.Path = this.current_board.path
      }

      return record
    },
    isLinuxFlightController(): boolean {
      // this is setup this way so we can include other linux boards in the list in the future
      const boardname = this.current_board?.name
      if (!boardname) {
        return false
      }
      return ['Navigator', 'SITL'].includes(boardname)
    },
    current_board(): FlightController | null {
      return autopilot.current_board
    },
    firmware_info(): FirmwareInfo | null {
      return autopilot.firmware_info
    },
    vehicle_type(): string | null {
      return autopilot.vehicle_type
    },
    board_undefined(): boolean {
      return this.current_board === null
    },
    restarting(): boolean {
      return autopilot.restarting
    },
  },
  mounted() {
    callPeriodically(fetchAvailableBoards, 5000)
    callPeriodically(fetchCurrentBoard, 5000)
    callPeriodically(fetchFirmwareInfo, 5000)
    callPeriodically(fetchVehicleType, 5000)
  },
  beforeDestroy() {
    stopCallingPeriodically(fetchAvailableBoards)
    stopCallingPeriodically(fetchCurrentBoard)
    stopCallingPeriodically(fetchFirmwareInfo)
    stopCallingPeriodically(fetchVehicleType)
  },
  methods: {
    async start_autopilot(): Promise<void> {
      autopilot.setRestarting(true)
      await back_axios({
        method: 'post',
        url: `${autopilot.API_URL}/start`,
        timeout: 10000,
      })
        .catch((error) => {
          notifier.pushBackError('AUTOPILOT_START_FAIL', error)
        })
        .finally(() => {
          autopilot.setRestarting(false)
        })
    },
    async stop_autopilot(): Promise<void> {
      autopilot.setRestarting(true)
      await back_axios({
        method: 'post',
        url: `${autopilot.API_URL}/stop`,
        timeout: 10000,
      })
        .catch((error) => {
          notifier.pushBackError('AUTOPILOT_STOP_FAIL', error)
        })
        .finally(() => {
          autopilot.setRestarting(false)
        })
    },
    async restart_autopilot(): Promise<void> {
      autopilot_data.reset()
      autopilot.setRestarting(true)
      await back_axios({
        method: 'post',
        url: `${autopilot.API_URL}/restart`,
        timeout: 10000,
      })
        .catch((error) => {
          notifier.pushBackError('AUTOPILOT_RESTART_FAIL', error)
        })
        .finally(() => {
          autopilot.setRestarting(false)
        })
    },
    openBoardChangeDialog(): void {
      this.show_board_change_dialog = true
    },
  },
})
</script>
