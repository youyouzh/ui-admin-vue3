<template>
  <el-select
    v-model="modelValue"
    clearable
    filterable
    placeholder="请选择部署项目"
    :loading="optionLoading"
    :multiple="props.multi"
    :disabled="props.disabled"
    @change="handleChangeEvent"
  >
    <el-option
      v-for="item in options"
      :key="item.id"
      :label="`${item.code}(${item.name})`"
      :value="item.id"
    />
  </el-select>
</template>
<script setup lang="tsx">
import { propTypes } from '@/utils/propTypes'
import { api, ProjectVO } from '@/api/resource/project'

const props = defineProps({
  modelValue: propTypes.oneOfType([Number, Array<Number>]),
  multi: propTypes.bool.def(false),
  filterIds: propTypes.array.def([]),
  disabled: propTypes.bool.def(false)
})

const modelValue = ref(props.modelValue)
const emit = defineEmits(['update:modelValue'])

const optionLoading = ref(false)
const options = ref<ProjectVO[]>([])

const initSelectOptions = async () => {
  try {
    optionLoading.value = true
    const data = await api.getAll()
    options.value = data.filter((v) => !props.filterIds.includes(v.id))
  } finally {
    optionLoading.value = false
  }
}

const handleChangeEvent = () => {
  emit('update:modelValue', modelValue)
}

onMounted(() => {
  initSelectOptions()
})
</script>
