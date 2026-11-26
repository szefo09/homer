<template>
  <Generic :item="item">
    <template #subtitle>
      <div class="netgear-layout">
        <div class="netgear-grid">
          <div
            v-for="port in portList"
            :key="port.number"
            class="netgear-port"
            :title="`Port ${port.number}: ${port.speedLabel}`"
          >
            <span>{{ port.number }}</span>
            <span class="netgear-dot" :class="port.tone"></span>
          </div>
        </div>

        <div class="netgear-auto-reset">
          <span class="netgear-auto-label">AUTO</span>

          <button
            class="netgear-toggle"
            :class="{
              enabled: autoResetPort1 === true,
              disabled: autoResetPort1 === false
            }"
            :disabled="autoResetPort1 === null || autoResetBusy"
            :aria-pressed="autoResetPort1 === true"
            :title="autoResetPort1 === true
              ? 'Wyłącz automatyczny reset Portu 1'
              : 'Włącz automatyczny reset Portu 1'"
            @click="toggleAutoReset"
          >
            <span class="netgear-toggle-knob"></span>
          </button>

          <span
            class="netgear-auto-state"
            :class="{
              on: autoResetPort1 === true,
              off: autoResetPort1 === false
            }"
          >
            {{ autoResetPort1 === true ? "ON" :
               autoResetPort1 === false ? "OFF" : "?" }}
          </span>
        </div>
      </div>
    </template>
  </Generic>
</template>

<script>
import Generic from "./Generic.vue";
import service from "@/mixins/service.js";

export default {
  name: "Netgear",
  mixins: [service],

  components: {
    Generic,
  },

  data: () => ({
    ports: null,
    updatedAt: null,
    serverError: null,
    autoResetPort1: null,
    autoResetBusy: false,
  }),

  computed: {
    portList() {
      return Array.from({ length: 8 }, (_, index) => {
        const number = index + 1;
        const port = this.ports?.[String(number)];

        if (
          this.serverError === true ||
          !port ||
          !["on", "off"].includes(port.status)
        ) {
          return {
            number,
            tone: "info",
            speedLabel: "UNKNOWN",
          };
        }

        if (port.status === "off") {
          return {
            number,
            tone: "danger",
            speedLabel: "OFF",
          };
        }

        const speed = Number(port.connection_speed);

        if (speed === 1000) {
          return {
            number,
            tone: "success",
            speedLabel: "1000 Mbps",
          };
        }

        if (speed === 100 || speed === 10) {
          return {
            number,
            tone: "warning",
            speedLabel: `${speed} Mbps`,
          };
        }

        return {
          number,
          tone: "info",
          speedLabel: "UNKNOWN",
        };
      });
    },
  },

  methods: {
    fetchData() {
      return this.load(
        this.fetch("status.json", { cache: "no-store" })
          .then((data) => {
            if (!data || !data.ports) {
              throw new Error("Nieprawidłowa odpowiedź NETGEAR API");
            }

            for (let number = 1; number <= 8; number++) {
              const port = data.ports[String(number)];

              if (
                !port ||
                !["on", "off"].includes(port.status) ||
                !Number.isFinite(Number(port.connection_speed))
              ) {
                throw new Error(`Nieprawidłowe dane portu ${number}`);
              }
            }

            this.ports = data.ports;
            this.updatedAt = data.updated_at || null;

            this.autoResetPort1 =
              typeof data.auto_reset_port1 === "boolean"
                ? data.auto_reset_port1
                : null;

            this.serverError = false;
          })
          .catch((error) => {
            this.ports = null;
            this.serverError = true;
            throw error;
          }),
      );
    },

    async toggleAutoReset() {
      if (this.autoResetBusy || this.autoResetPort1 === null) {
        return;
      }

      this.autoResetBusy = true;

      try {
        const response = await fetch("/netgear/api/toggle", {
          method: "POST",
          cache: "no-store",
        });

        if (!response.ok) {
          throw new Error("Błąd przełączania auto-reset");
        }

        const data = await response.json();

        if (typeof data.auto_reset_port1 !== "boolean") {
          throw new Error("Nieprawidłowa odpowiedź API");
        }

        this.autoResetPort1 = data.auto_reset_port1;
      } catch (error) {
        console.error("Netgear toggle:", error);
      } finally {
        this.autoResetBusy = false;
      }
    },
  },
};
</script>

<style scoped>
.netgear-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  align-items: center;
  gap: 8px;
}

.netgear-grid {
  display: grid;
  width: 95%;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  grid-auto-rows: clamp(18px, 24vw, 24px);
  gap: 4px;
}

.netgear-port {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  min-width: 0;
  font-size: 0.65rem;
  font-weight: 600;
  border: 1px solid rgba(127, 127, 127, 0.18);
  border-radius: 3px;
  background: rgba(127, 127, 127, 0.05);
}

.netgear-dot {
  display: inline-block;
  width: 7px;
  height: 7px;
  min-width: 7px;
  min-height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
}

.netgear-dot.success {
  background-color: var(--status-online);
  box-shadow: 0 0 2px 1px var(--status-online);
}

.netgear-dot.warning {
  background-color: var(--status-warning);
  box-shadow: 0 0 2px 1px var(--status-warning);
}

.netgear-dot.danger {
  background-color: var(--status-offline);
}

.netgear-dot.info {
  background-color: var(--status-unknown);
  box-shadow: 0 0 2px 1px var(--status-unknown);
}

.netgear-auto-reset {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 2px;
  min-width: 34px;
}

.netgear-auto-label {
  font-size: 0.55rem;
  font-weight: 700;
  opacity: 0.65;
}
.netgear-toggle {
  display: flex;
  align-items: center;
  width: 30px;
  height: 17px;
  padding: 2px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  transition: background-color 0.2s;
  flex-shrink: 0;
}

.netgear-toggle.enabled {
  background: var(--status-online);
}

.netgear-toggle.disabled {
  background: var(--status-unknown);
}

.netgear-toggle-knob {
  display: block;
  width: 13px;
  height: 13px;
  border-radius: 50%;
  background: white;
  transition: transform 0.2s;
  flex-shrink: 0;
}

.netgear-toggle.enabled .netgear-toggle-knob {
  transform: translateX(13px);
}

.netgear-toggle:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.netgear-auto-state {
  font-size: 0.55rem;
  font-weight: 700;
}

.netgear-auto-state.on {
  color: var(--status-online);
}

.netgear-auto-state.off {
  color: var(--status-unknown);
}
</style>
