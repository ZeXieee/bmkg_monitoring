<script setup lang="ts">
import { ref, computed, onMounted } from "vue";

// Ganti adm4 kalau mau lokasi lain
const API_URL =
  "https://api.bmkg.go.id/publik/prakiraan-cuaca?adm4=36.03.18.2004";

type DataPoint = {
  timestamp: string; // ISO-like dari local_datetime
  temperature: number;
  humidity: number;
  windSpeed: number;
  condition: string;
  iconUrl?: string;
};

type Lokasi = {
  provinsi: string;
  kotkab: string;
  kecamatan: string;
  desa: string;
};

const loading = ref(true);
const error = ref<string | null>(null);
const points = ref<DataPoint[]>([]);
const lokasi = ref<Lokasi | null>(null);
const analysisDate = ref<string | null>(null);

async function load() {
  loading.value = true;
  error.value = null;
  try {
    const res = await fetch(API_URL);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const json = await res.json();

    // Lokasi utama
    if (json.lokasi) {
      lokasi.value = {
        provinsi: json.lokasi.provinsi,
        kotkab: json.lokasi.kotkab,
        kecamatan: json.lokasi.kecamatan,
        desa: json.lokasi.desa,
      };
    }

    // Flatten cuaca 3D → 1D
    const cuacaNested: any[][] =
      json.data && json.data[0] && Array.isArray(json.data[0].cuaca)
        ? json.data[0].cuaca
        : [];

    const flat: any[] = ([] as any[]).concat(...cuacaNested);

    // Ambil analysis_date pertama (UTC, tanpa zona → anggap Z)
    const firstWithAnalysis = flat.find((x) => x.analysis_date);
    if (firstWithAnalysis?.analysis_date) {
      const raw: string = firstWithAnalysis.analysis_date;
      analysisDate.value = raw.endsWith("Z") ? raw : `${raw}Z`;
    }

    // Map ke DataPoint
    points.value = flat.map((item) => {
      const rawLocal: string = item.local_datetime; // "YYYY-MM-DD HH:mm:ss"
      const isoLocal = rawLocal.replace(" ", "T") + "+07:00";

      return {
        timestamp: isoLocal,
        temperature: Number(item.t),
        humidity: Number(item.hu),
        windSpeed: Number(item.ws),
        condition: String(item.weather_desc ?? ""),
        iconUrl: item.image as string | undefined,
      };
    });

    // Urutkan waktu naik
    points.value.sort(
      (a, b) =>
        new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
    );
  } catch (e: any) {
    console.error(e);
    error.value = e?.message ?? "Unknown error";
  } finally {
    loading.value = false;
  }
}

// ---------- Computed ----------

const lokasiText = computed(() => {
  if (!lokasi.value) return "–";
  const l = lokasi.value;
  return `${l.desa}, ${l.kecamatan}, ${l.kotkab}, ${l.provinsi}`;
});

const lastPoint = computed<DataPoint | null>(() => {
  const arr = points.value;
  if (!arr || arr.length === 0) {
    return null;
  }
  // "!" di sini bilang ke TypeScript: ini pasti ada (bukan undefined)
  return arr[arr.length - 1]!;
});


const last24h = computed<DataPoint[]>(() => {
  if (!points.value.length) return [];
  const cutoff = new Date();
  cutoff.setHours(cutoff.getHours() - 24);
  return points.value.filter(
    (p) => new Date(p.timestamp).getTime() >= cutoff.getTime()
  );
});

function avg(arr: number[]) {
  if (!arr.length) return NaN;
  return arr.reduce((a, b) => a + b, 0) / arr.length;
}

const avg24h = computed(() => {
  const arr = last24h.value;
  return {
    temperature: avg(arr.map((p) => p.temperature)),
    humidity: avg(arr.map((p) => p.humidity)),
    windSpeed: avg(arr.map((p) => p.windSpeed)),
  };
});

// ---------- Formatter ----------

function formatDate(dt: string | null) {
  if (!dt) return "–";
  const d = new Date(dt);
  return d.toLocaleString("id-ID", {
    day: "2-digit",
    month: "short",
    year: "numeric",
    hour: "2-digit",
    minute: "2-digit",
  });
}

function formatHour(dt: string) {
  const d = new Date(dt);
  return d.toLocaleTimeString("id-ID", {
    hour: "2-digit",
    minute: "2-digit",
  });
}

function fmt(value: number, digits: number) {
  if (Number.isNaN(value)) return "–";
  return value.toFixed(digits);
}

onMounted(load);
</script>

<template>
  <div class="page">
    <!-- HERO -->
    <header class="hero animate-fade">
      <div class="hero-text">
        <p class="badge">BMKG · Talaga, Cikupa</p>
        <h1>Monitoring Cuaca</h1>
        <p class="subtitle">
          dari API BMKG, diperbarui setiap 3 jam.
        </p>
      </div>

      <div class="hero-badge">
        <span class="hero-badge-label">Lokasi</span>
        <span class="hero-badge-main">
          {{ lokasiText }}
        </span>
        <span class="hero-badge-label">Update terakhir (BMKG)</span>
        <span class="hero-badge-time">
          {{ formatDate(analysisDate) }}
        </span>
      </div>
    </header>

    <!-- STATUS (hanya loading & error) -->
    <section class="status-row">
      <div v-if="loading" class="status-chip loading">Mengambil data…</div>
      <div v-else-if="error" class="status-chip error">
        Gagal memuat data: {{ error }}
      </div>
    </section>

    <!-- CARDS -->
    <section class="cards-grid animate-rise" v-if="lastPoint">
      <article class="card">
        <h2>Suhu</h2>
        <p class="card-value">
          {{ lastPoint.temperature.toFixed(1) }}<span class="unit">°C</span>
        </p>
        <p class="card-sub">
          Rata-rata 24 jam: {{ fmt(avg24h.temperature, 1) }}°C
        </p>
      </article>

      <article class="card">
        <h2>Kelembaban Udara</h2>
        <p class="card-value">
          {{ lastPoint.humidity.toFixed(0) }}<span class="unit">%</span>
        </p>
        <p class="card-sub">
          Rata-rata 24 jam: {{ fmt(avg24h.humidity, 0) }}%
        </p>
      </article>

      <article class="card">
        <h2>Kecepatan Angin</h2>
        <p class="card-value">
          {{ lastPoint.windSpeed.toFixed(1) }}<span class="unit">
            km/jam
          </span>
        </p>
        <p class="card-sub">
          Rata-rata 24 jam: {{ fmt(avg24h.windSpeed, 1) }} km/jam
        </p>
      </article>

      <article class="card card-condition">
        <h2>Kondisi Cuaca</h2>
        <div class="condition-inner">
          <img
            v-if="lastPoint.iconUrl"
            :src="lastPoint.iconUrl"
            alt="Ikon cuaca"
            class="condition-icon"
          />
          <p class="card-value-small">
            {{ lastPoint.condition || "–" }}
          </p>
        </div>
      </article>
    </section>

    <!-- 24 JAM TERAKHIR -->
    <section class="panel animate-rise-late">
      <header class="panel-header">
        <h2>Data selama 24 jam terakhir</h2>
        <p>(Perkiraan sampai 3 hari kedepan)</p>
      </header>

      <div class="table-wrapper">
        <table v-if="last24h.length">
          <thead>
            <tr>
              <th>No</th>
              <th>Waktu</th>
              <th>Suhu (°C)</th>
              <th>Kelembaban (%)</th>
              <th>Kecepatan Angin (km/jam)</th>
              <th>Kondisi</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(p, idx) in last24h" :key="p.timestamp">
              <td>{{ idx + 1 }}</td>
              <td>{{ formatHour(p.timestamp) }}</td>
              <td>{{ p.temperature.toFixed(1) }}</td>
              <td>{{ p.humidity.toFixed(0) }}</td>
              <td>{{ p.windSpeed.toFixed(1) }}</td>
              <td>{{ p.condition }}</td>
            </tr>
          </tbody>
        </table>

        <p v-else class="empty-text">
          Belum ada data 24 jam terakhir.
        </p>
      </div>
    </section>
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  padding: 2.5rem 1.25rem 3rem;
  max-width: 1120px;
  margin: 0 auto;
  background: radial-gradient(circle at top, #1f2937 0, #020617 55%);
  color: #e5e7eb;
  /* Center all table cells */
  table th,
  table td {
    text-align: center !important;
  }

}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.animate-fade {
  animation: fadeIn 0.6s ease-out;
}

.animate-rise {
  animation: fadeIn 0.7s ease-out 0.1s backwards;
}

.animate-rise-late {
  animation: fadeIn 0.7s ease-out 0.2s backwards;
}

.hero {
  display: flex;
  gap: 1.75rem;
  align-items: stretch;
  justify-content: space-between;
  margin-bottom: 1.75rem;
  flex-wrap: wrap;
}

.hero-text {
  flex: 1 1 260px;
}

.badge {
  display: inline-flex;
  padding: 0.15rem 0.65rem;
  border-radius: 999px;
  border: 1px solid rgba(148, 163, 184, 0.5);
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: #9ca3af;
  margin-bottom: 0.6rem;
  background: rgba(15, 23, 42, 0.9);
}

.hero h1 {
  font-size: clamp(1.9rem, 2.3vw + 1.2rem, 2.4rem);
  font-weight: 700;
  margin-bottom: 0.35rem;
}

.subtitle {
  color: #9ca3af;
  max-width: 34rem;
  font-size: 0.95rem;
  line-height: 1.55;
}

.hero-badge {
  flex: 0 0 230px;
  align-self: flex-start;
  padding: 0.75rem 0.85rem;
  background: radial-gradient(circle at top, #111827 0, #020617 65%);
  border-radius: 1rem;
  border: 1px solid rgba(148, 163, 184, 0.5);
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
  font-size: 0.75rem;
}

.hero-badge-label {
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: #9ca3af;
}

.hero-badge-main {
  font-weight: 600;
}

.hero-badge-time {
  font-weight: 600;
  color: #e5e7eb;
}

/* STATUS CHIP */

.status-row {
  margin-bottom: 1.4rem;
}

.status-chip {
  display: inline-flex;
  align-items: center;
  padding: 0.45rem 0.9rem;
  border-radius: 999px;
  font-size: 0.8rem;
  gap: 0.35rem;
}

.status-chip.loading {
  background: rgba(59, 130, 246, 0.18);
  border: 1px solid rgba(59, 130, 246, 0.6);
  color: #dbeafe;
}

.status-chip.error {
  background: rgba(239, 68, 68, 0.18);
  border: 1px solid rgba(239, 68, 68, 0.7);
  color: #fee2e2;
}

/* CARDS */

.cards-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 1rem;
  margin-bottom: 1.75rem;
}

.card {
  background: radial-gradient(circle at top left, #0f172a 0, #020617 75%);
  border-radius: 1.1rem;
  padding: 1.1rem 1rem;
  border: 1px solid rgba(148, 163, 184, 0.32);
  box-shadow: 0 20px 50px rgba(15, 23, 42, 0.7);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;       /* rata tengah */
  text-align: center;        /* rata tengah */
  transition: transform 0.18s ease-out, box-shadow 0.18s ease-out,
    border-color 0.18s ease-out;
}

.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 26px 70px rgba(30, 64, 175, 0.55);
  border-color: rgba(129, 140, 248, 0.7);
}

.card h2 {
  font-size: 0.9rem;
  font-weight: 500;
  color: #9ca3af;
  margin-bottom: 0.45rem;
}

.card-value {
  font-size: 1.7rem;
  font-weight: 700;
}

.card-value .unit {
  font-size: 0.9rem;
  color: #9ca3af;
  margin-left: 0.2rem;
}

.card-sub {
  margin-top: 0.45rem;
  font-size: 0.8rem;
  color: #9ca3af;
}

.card-condition .condition-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.35rem;
  margin-top: 0.2rem;
}

.condition-icon {
  width: 42px;
  height: 42px;
}

.card-value-small {
  font-size: 1rem;
  font-weight: 600;
  text-align: center;
}

/* PANEL + TABLE */

.panel {
  background: rgba(15, 23, 42, 0.96);
  border-radius: 1.2rem;
  border: 1px solid rgba(148, 163, 184, 0.33);
  padding: 1.2rem 1.1rem 1.1rem;
}

.panel-header {
  text-align: center;
  margin-bottom: 0.9rem;
}

.panel-header h2 {
  font-size: 1.05rem;
  font-weight: 600;
  margin-bottom: 0.2rem;
}

.panel-header p {
  font-size: 0.8rem;
  color: #9ca3af;
}

.table-wrapper {
  max-height: 360px;
  overflow: auto;
  border-radius: 0.9rem;
  border: 1px solid rgba(55, 65, 81, 0.9);
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.84rem;
}

thead {
  position: sticky;
  top: 0;
  background: #020617;
  z-index: 1;
}

th,
td {
  padding: 0.55rem 0.75rem;
  text-align: left;
}

th {
  font-weight: 500;
  color: #9ca3af;
  border-bottom: 1px solid rgba(55, 65, 81, 0.95);
  white-space: nowrap;
}

tbody tr:nth-child(odd) {
  background: rgba(15, 23, 42, 0.96);
}

tbody tr:nth-child(even) {
  background: rgba(17, 24, 39, 0.96);
}

tbody tr:hover {
  background: rgba(30, 64, 175, 0.5);
}

.empty-text {
  padding: 0.9rem 0.75rem;
  font-size: 0.85rem;
  color: #9ca3af;
}

/* RESPONSIVE */

@media (max-width: 900px) {
  .hero {
    flex-direction: column;
  }

  .hero-badge {
    flex: 1 1 auto;
  }

  .cards-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 640px) {
  .page {
    padding-inline: 1rem;
  }

  .hero {
    gap: 1.1rem;
  }

  .hero-badge {
    font-size: 0.74rem;
  }

  .cards-grid {
    grid-template-columns: minmax(0, 1fr);
  }

  th,
  td {
    padding-inline: 0.5rem;
  }
}
</style>
