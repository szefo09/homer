
<template>
  <Generic :item="item">
    <template #subtitle>
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
  }),

  computed: {
portList() {
  return Array.from({ length: 8 }, (_, index) => {
    const number = index + 1;
    const port = this.ports?.[String(number)];

    // Brak danych lub błąd odczytu
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

    // Port fizycznie wyłączony
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
        this.serverError = false;
      })
      .catch((error) => {
        this.ports = null;
        this.serverError = true;
        throw error;
      }),
  );
},
  },
};
</script>

<style scoped>
.netgear-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 5px;
}

.netgear-port {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  height: 22px;
  font-size: 0.65rem;
  font-weight: 600;
  border: 1px solid rgba(127, 127, 127, 0.18);
  border-radius: 3px;
  background: rgba(127, 127, 127, 0.05);
}

.netgear-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  min-width: 8px;
  min-height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
  vertical-align: middle;
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
</style>