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
                <div class="container p-8 mx-auto">
                    <input type="checkbox" id="form_modal" class="modal-toggle" />
                    <div class="modal" role="dialog">
                        <form @submit="createData()" class="modal-box space-y-3">
                            <h3 class="text-lg font-bold mb-4">Manage Scheduling</h3>
                            <div>
                                <label for="days">Days</label>
                                <div class="grid-cols-7 grid gap-3">
                                    <button type="button" :class="{'bg-primary text-white': selectedDays.includes(index)}"  @click="selectingDay(index)" v-for="(item, index) in days" :key="index" class="btn border-primary">{{ item }}</button>
                                </div>
                            </div>
                            <div class="grid grid-cols-2 gap-3">
                                <div class="form-control">
                                    <label for="time" class="capitalize">time</label>
                                    <input id="time" type="time" class="input input-bordered w-full" v-model="time">
                                </div>
                                <div class="form-control">
                                    <label for="duration" class="capitalize">duration (minutes)</label>
                                    <input id="duration" type="number" class="input input-bordered w-full" v-model="duration">
                                </div>
                            </div>
                            <div class="form-control">
                                <label for="actuator" class="capitalize">actuator (rpm)</label>
                                <input id="actuator" type="number" class="input input-bordered w-full" v-model="actuator">
                            </div>
                            <div class="flex items-center justify-end w-full gap-3">
                                <label for="form_modal" class="btn btn-base-300">Cancel</label>
                                <button type="submit" class="btn btn-primary">Save</button>
                            </div>
                        </form>
                        <label class="modal-backdrop" for="form_modal">Close</label>
                    </div>
                    <div class="flex items-center gap-3">
                        <label for="form_modal" class="btn btn-primary">Add Data</label>
                        <button @click="deleteAll" class="btn btn-error">Delete All</button>
                    </div>
                    <div class="grid lg:grid-cols-6 md:grid-cols-2 grid-cols-1 gap-3 mt-4">
                        <div v-for="(item, index) in scheduleData" :key="index" class="">
                            <div class="w-full p-4 rounded-xl bg-base-100">
                                <div class="flex flex-wrap gap-1 mt-2 mb-4">
                                    <span v-for="day in getDayNamesArray(item.days)" :key="day" class="badge bg-primary text-white px-2 py-1 rounded-lg text-xs">
                                        {{ day }}
                                    </span>
                                </div>
                                <p>Time: {{ item.time }}</p>
                                <p>Duration: {{ item.duration }} minutes</p>
                                <p>Actuator: {{ item.actuator }} Rpm</p>
                                <div class="grid-cols-2 grid gap-3 mt-3">
                                    <button @click="editData(item)" class="btn btn-primary btn-sm">Edit</button>
                                    <button @click="deleteByKey(item.key)" class="btn btn-error btn-sm">Delete</button>

                                </div>
                            </div>
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
import { push, set, remove } from 'firebase/database';

onMounted(() => {
  fetchDataFromFirebase();
  document.documentElement.setAttribute('data-theme', 'light')
});

const scheduleData: Ref<any> = ref([])
const actuator: Ref<number> = ref(120)
const duration: Ref<number> = ref(1)
const time: Ref<string> = ref('');
const days: Ref<any> = ref(["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"])
const daysList = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
const currentKey: Ref<string | null> = ref(null);

const selectedDays: Ref<any> = ref([])

function getDayNamesArray(days: string) {
    return days
        .split(',')
        .map(day => daysList[parseInt(day, 10)]); // Convert each number to day name
}

function selectingDay(idx: number) {
    const dayIndex = selectedDays.value.indexOf(idx);
    if (dayIndex !== -1) {
        selectedDays.value.splice(dayIndex, 1);
    } else {
        selectedDays.value.push(idx);
    }
    console.log(selectedDays.value);
}

async function fetchDataFromFirebase() {
    try {
        const snapshot = await get(firebaseRef(database, 'love_bird/schedule'));
        if (snapshot.exists()) {
            const data = snapshot.val();
            scheduleData.value = Object.entries(data).map(([key, value]: any) => ({
                key,
                actuator: value.actuator,
                days: value.days,
                duration: value.duration,
                time: value.time,
            }));
            console.log(scheduleData.value)
        } else {
            console.log("No data available");
        }
    } catch (error) {
        console.error("Error fetching data from Firebase:", error);
    }
}

function editData(item: any) {
    currentKey.value = item.key;
    actuator.value = item.actuator;
    duration.value = item.duration;
    time.value = item.time;
    selectedDays.value = item.days.split(',').map(Number);
    (document.getElementById('form_modal') as HTMLInputElement).checked = true;
}

async function createData() {
    try {
        const daysString = selectedDays.value.join(','); // Convert selectedDays to a string format
        const newData = {
            actuator: actuator.value,
            days: daysString,
            duration: duration.value,
            time: time.value
        };

        if (currentKey.value) {
            await set(firebaseRef(database, `love_bird/schedule/${currentKey.value}`), newData);
            console.log("Data updated successfully");
        } else {
            await push(firebaseRef(database, 'love_bird/schedule'), newData);
            console.log("Data added successfully");
        }
        
        await fetchDataFromFirebase();

        actuator.value = 120;
        duration.value = 1;
        time.value = '';
        selectedDays.value = [];
    } catch (error) {
        console.error("Error adding data to Firebase:", error);
    }
}

async function deleteByKey(key: string) {
    try {
        await remove(firebaseRef(database, `love_bird/schedule/${key}`));
        console.log(`Data with key ${key} deleted successfully`);
        await fetchDataFromFirebase();
    } catch (error) {
        console.error(`Error deleting data with key ${key}:`, error);
    }
}

async function deleteAll() {
    try {
        await remove(firebaseRef(database, 'love_bird/schedule'));
        console.log("All data deleted successfully");
        scheduleData.value = [];
    } catch (error) {
        console.error("Error deleting all data:", error);
    }
}
</script>