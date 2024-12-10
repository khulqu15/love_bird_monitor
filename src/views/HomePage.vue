<template>
  <ion-page>
    <ion-content :fullscreen="true">
      <div class="min-h-screen w-full relative bg-base-200 text-base-content">
        <div class="py-2 px-4 bg-base-100 w-full flex items-center justify-between">
          <div class="w-full">
            <h2>Love Bird</h2>
          </div>
          <div class="flex items-center gap-3">
            <button class="btn btn-base-300" @click="$router.push({name: 'Home'})">Home</button>
            <button class="btn btn-base-300" @click="$router.push({name: 'Scheduling'})">Scheduling</button>
          </div>
        </div>
        <div class="grid grid-cols-4 min-h-[88vh] items-center justify-items-center">
          <div class="col-span-4 md:col-span-2 p-4 text-left w-full space-y-2">
            <img src="/public/banner.png" class="w-full shadow-xl rounded-3xl" alt="">
            <card-view-vue header="Data Table">
              <div class="flex items-center gap-3 mb-6">
                <button class="btn btn-primary" @click="exportToExcel()">Export Excel</button>
                <button class="btn btn-error" @click="deleteAll()">Delete All</button>
              </div>
              <div class="w-full max-h-[60vh] h-[20vh] overflow-auto">
                <table class="table table-sm">
                  <thead>
                    <tr>
                      <th></th>
                      <th>Load Cell</th>
                      <th>Water Level</th>
                      <th>Timestamp</th>
                      <th>Action</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="(item, index) in tableData" :key="index">
                      <th>{{ index + 1 }}</th>
                      <td>{{ item.loadcell }} gram</td>
                      <td>{{ item.water_level }} mm</td>
                      <td>{{ item.timestamp }}</td>
                      <td>
                        <button @click="deleteByKey(item.key)" class="btn btn-error btn-sm">Delete</button>
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </card-view-vue>
          </div>

          <div class="col-span-4 md:col-span-2 p-4 text-left w-full space-y-2">
            <div>
              <div class="p-3 rounded-xl gap-4 bg-base-100 flex items-center">
                <label for="servo_value" class="font-bold">Servo value</label>
                <div class="w-full">
                  <input @keyup="changeData()" id="servo_value" v-model="servoValue.degree" class="input input-bordered w-full">
                </div>
                <button @click="toggleRun()" class="btn text-white" :class="{'btn-error': servoValue.active == false, 'btn-primary': servoValue.active == true}">
                  {{ !servoValue.active ? 'Stop' : 'Run' }}
                </button>
              </div>
              <div class="grid grid-cols-2 p-4 rounded-xl bg-base-100 mt-2">
                <div v-for="(item, index) in 4" :key="index" class=" justify-between">
                  <div class="flex items-center gap-3">
                    Relay {{ item }}
                    <input @change="changeRelay(index)" type="checkbox" class="toggle toggle-primary" />
                  </div>
                </div>
              </div>
            </div>
            <div v-if="waves.length > 0">
              <card-view-vue header="Chart Plotting">
                <div v-if="selectedWave == -1">
                  <waves-chart-vue 
                    :wave-data="waves.map(wave => wave.data)" 
                    :wave-names="waves.map(wave => wave.name)" 
                  />
                </div>
                <div v-else>
                  <waves-chart-vue 
                    :wave-data="[waves[selectedWave].data]" 
                    :wave-names="[waves[selectedWave].name]" 
                  />
                </div>
              </card-view-vue>
            </div>
          </div>

        </div>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import CardViewVue from '@/components/CardView.vue';
import { IonContent, IonPage } from '@ionic/vue';
import { ref, Ref, onMounted } from 'vue';
import { database, ref as firebaseRef, get } from '@/firebaseConfig';
import { remove, child, onValue, set } from 'firebase/database';
import WavesChartVue from '@/components/WavesChart.vue';
import * as XLSX from 'xlsx'

const selectedWave: Ref<any> = ref(-1);
const tableData: Ref<any> = ref([])
const relayValue: Ref<any> = ref({
  0: false,
  1: false,
  2: false,
  3: false
});
const servoValue: Ref<any> = ref({});

onMounted(() => {
  fetchDataFromFirebase();
  document.documentElement.setAttribute('data-theme', 'light')
});

async function fetchDataFromFirebase() {
  try {
    const relayRef = firebaseRef(database, 'love_bird/relay');
    onValue(relayRef, (snapshot: any) => {
      if (snapshot.exists()) {
        relayValue.value = snapshot.val();
        console.log(relayValue.value);
      } else {
        console.log("No relay value available");
      }
    }, (error: any) => {
      console.error("Error fetching relay value from Firebase:", error);
    });

    const servoRef = firebaseRef(database, 'love_bird/servo');
    onValue(servoRef, (snapshot: any) => {
      if (snapshot.exists()) {
        servoValue.value = snapshot.val();
      } else {
        console.log("No servo value available");
      }
    }, (error: any) => {
      console.error("Error fetching servo value from Firebase:", error);
    });

    const dataRef = firebaseRef(database, 'love_bird/data');
    onValue(dataRef, (snapshot: any) => {

      if (snapshot.exists()) {
        const data = snapshot.val();
        tableData.value = Object.entries(data).map(([key, value]: any) => ({
          key,
          loadcell: value.load_cell,
          timestamp: value.timestamp,
          water_level: value.water_level
        }));

        const loadCells = [];
        const waterLevels = [];

        for (const key in data) {
          const entry = data[key];
          const [datePart, timePart] = entry.timestamp.split(" ");
          const [day, month, year] = datePart.split("/");
          const formattedDate = `${year}-${month}-${day}T${timePart}:00`;

          const date = new Date(formattedDate);
          if (!isNaN(date.getTime())) {
            loadCells.push({
              value: entry.load_cell,
              date: formattedDate
            });

            waterLevels.push({
              value: entry.water_level,
              date: formattedDate
            });
          } else {
            console.error("Invalid date:", entry.timestamp);
          }
        }

        waves.value[0].data = loadCells;
        waves.value[1].data = waterLevels;
      } else {
        console.log("No data available");
      }
    }, (error: any) => {
      console.error("Error fetching data from Firebase:", error);
    });
  } catch (error) {
    console.error("Error initializing Firebase listener:", error);
  }
}

function toggleRun() {
  servoValue.value.active = !servoValue.value.active;
  set(firebaseRef(database, 'love_bird/servo'), servoValue.value);
}

function changeData() {
  set(firebaseRef(database, 'love_bird/servo'), {
    degree: parseInt(servoValue.value.degree),
    active: servoValue.value.active
  });
}

const waves = ref([
  { name: 'Load Cell', data: [] as { value: number; date: string }[] },
  { name: 'Water Level', data: [] as { value: number; date: string }[] },
]);


async function deleteByKey(key: string) {
  try {
    await remove(firebaseRef(database, `love_bird/data/${key}`));
    tableData.value = tableData.value.filter((item: any) => item.key !== key);
    console.log(`Entry with key ${key} deleted successfully`);
  } catch (error) {
    console.error(`Error deleting entry with key ${key}:`, error);
  }
}

async function deleteAll() {
  try {
    await remove(firebaseRef(database, 'love_bird/data'));
    tableData.value = [];
    console.log("All entries deleted successfully");
  } catch (error) {
    console.error("Error deleting all entries:", error);
  }
}

function changeRelay(index: number) {
  set(firebaseRef(database, `love_bird/relay/${index}`), !relayValue.value[index]);
  set(firebaseRef(database, `love_bird/relay/"${index}"`), !relayValue.value[index]);
}

function exportToExcel() {
  const waveData = waves.value.map(wave => {
    return {
      name: wave.name,
      data: wave.data.map(point => ({
        value: point.value,
        date: point.date
      }))
    };
  });

  const waveSheet = XLSX.utils.json_to_sheet(
    waveData.flatMap(wave => wave.data.map((point, index) => ({
      Name: wave.name,
      Value: point.value,
      Date: point.date,
      Index: index + 1
    })))
  );

  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, waveSheet, 'Waves');
  XLSX.writeFile(workbook, 'WaveData.xlsx');
}

</script>