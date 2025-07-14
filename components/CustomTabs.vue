<template>
  <div v-if="djangoModels.length" class="custom-tabs-container">
    <!-- Pestañas -->
    <div class="flex border-b border-gray-200 overflow-x-auto py-4">
      <button
        v-for="(model, index) in djangoModels"
        :key="index"
        @click="activeTab = index"
        class="px-4 py-2 font-medium text-sm whitespace-nowrap"
        :class="{
          'border-b-2 border-green-500 text-green-600': activeTab === index,
          'text-gray-500 hover:text-green-700': activeTab !== index,
        }"
      >
        {{ model.modelName }}
        <UButton
          size="2xs"
          icon="i-heroicons-document-duplicate"
          @click.stop="copyModel(index)"
          color="gray"
          variant="ghost"
          class="ml-2"
        />
      </button>
    </div>

    <!-- Contenido de las pestañas -->
    <div class="tab-content mt-2">
      <div
        v-for="(model, index) in djangoModels"
        :key="index"
        v-show="activeTab === index"
        class="bg-white rounded-lg shadow overflow-hidden"
      >
        <UCard>
          <template #header>
            <div class="flex justify-between items-center">
              <h3 class="font-semibold">{{ model.modelName }}</h3>
              <UButton
                size="xs"
                icon="i-heroicons-document-duplicate"
                @click="copyModel(index)"
              />
            </div>
          </template>
          <CodeEditor v-model="model.modelCode" language="python"> </CodeEditor>
        </UCard>
      </div>
    </div>
  </div>
</template>

<script setup>
import "prismjs/plugins/line-numbers/prism-line-numbers.js";
import CodeEditor from "./CodeEditor.vue";
import VCodeBlock from "@wdns/vue-code-block";
const props = defineProps({
  djangoModels: {
    type: Array,
    required: true,
  },
});

const activeTab = ref(0);

const copyModel = async (index) => {
  try {
    await navigator.clipboard.writeText(props.djangoModels[index].modelCode);
    // Aquí podrías añadir una notificación de éxito
  } catch (err) {
    console.error("Error al copiar el modelo", err);
  }
};
</script>

<style scoped>
.custom-tabs-container {
  @apply w-full;
}

.custom-tabs-container .flex {
  @apply -mb-px; /* Para alinear con el borde inferior */
}

.custom-tabs-container button {
  @apply transition-colors duration-200 focus:outline-none;
}

.tab-content {
  @apply min-h-[300px];
}
</style>
