<template>
    <div class="w-full relative overflow-hidden bg-base-100 rounded-xl">
        <!-- <div class="w-full py-2 px-4 text-white">{{ header }}</div> -->
        <div class="p-3">
            <slot></slot>
        </div>
    </div>
</template>

<script lang="ts">
import { defineComponent, PropType } from "vue"

export default defineComponent ({
    props: {
        header: {
            type: String as PropType<string>,
            required: true,
        }
    }
})
</script>