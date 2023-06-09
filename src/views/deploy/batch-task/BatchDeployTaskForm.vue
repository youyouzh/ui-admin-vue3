<template>
  <Dialog v-model="dialogVisible" :title="dialogTitle + '批量部署任务'" width="800">
    <el-form
      ref="formRef"
      v-loading="formLoading"
      :model="formData"
      :rules="formRules"
      label-width="80px"
    >
      <el-form-item label="任务标题" prop="title">
        <el-input v-model="formData.title" placeholder="请输入任务标题" />
      </el-form-item>
      <el-form-item label="修改日志" prop="changeLog">
        <el-input
          v-model="formData.changeLog"
          type="textarea"
          placeholder="请输入修改日志"
          :autosize="{ minRows: 2 }"
        />
      </el-form-item>
      <el-form-item label="部署时间" prop="deployStartTime">
        <el-date-picker
          v-model="formData.deployStartTime"
          placeholder="请输入部署时间"
          type="datetime"
          :default-time="new Date()"
        />
      </el-form-item>
      <el-form-item label="部署服务" prop="deployTasks">
        <el-button type="primary" @click="handleAddDeployTask">
          <Icon icon="ep:plus" />添加部署服务
        </el-button>
        <el-table :data="formData.deployTasks">
          <el-table-column label="项目" align="center" prop="projectId">
            <template #default="scope">
              <ProjectSelect
                v-model="scope.row.projectId"
                :filter-ids="formData.deployTasks.map((v) => v.projectId)"
              />
            </template>
          </el-table-column>
          <el-table-column label="部署机器" align="center" prop="agentIds">
            <template #default="scope">
              <AgentSelect
                v-if="scope.row.projectId"
                v-model="scope.row.agentIds"
                :project-id="scope.row.projectId"
              />
            </template>
          </el-table-column>
          <el-table-column label="部署版本" align="center" prop="projectVersionId">
            <template #default="scope">
              <ProjectVersionSelect
                v-if="scope.row.projectId"
                v-model="scope.row.projectVersionId"
                :project-id="scope.row.projectId"
              />
            </template>
          </el-table-column>
          <el-table-column label="操作" align="center" fixed="right" width="80">
            <template #default="scope">
              <el-button
                link
                type="danger"
                @click="handleRemoveDeployTask(scope.row)"
                v-hasPermi="['resource:project:delete']"
              >
                <Icon icon="ep:close" />
              </el-button>
            </template>
          </el-table-column>
        </el-table>
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button type="primary" @click="submitForm">确 定</el-button>
      <el-button @click="dialogVisible = false">取 消</el-button>
    </template>
  </Dialog>
</template>
<script lang="ts" name="BatchDeployTaskForm" setup>
import { cloneDeep } from '@/utils'
import { api } from '@/api/deploy/batch-task'
import ProjectSelect from '@/views/resource/project/ProjectSelect.vue'
import AgentSelect from '@/views/resource/agent/AgentSelect.vue'
import ProjectVersionSelect from '@/views/resource/project/ProjectVersionSelect.vue'

const { t } = useI18n() // 国际化
const message = useMessage() // 消息弹窗

const dialogVisible = ref(false) // 弹窗的是否展示
const dialogTitle = ref('') // 弹窗的标题
const formLoading = ref(false) // 表单的加载中：1）修改时的数据加载；2）提交的按钮禁用
const formType = ref('') // 表单的类型：create - 新增；update - 修改

const defaultFormData = {
  id: undefined,
  title: undefined,
  changeLog: undefined,
  deployTasks: [],
  deployStartTime: new Date()
}
const formRef = ref()
const formData = ref(cloneDeep(defaultFormData))
const formRules = reactive({
  title: [{ required: true, message: '任务标题不能为空', trigger: 'blur' }],
  changeLog: [{ required: false, message: '部署版本不能为空', trigger: 'blur' }],
  deployTasks: [{ required: true, message: '部署服务不能为空', trigger: 'blur' }],
  deployStartTime: [{ required: true, message: '部署时间不能为空', trigger: 'blur' }]
})

/** 打开弹窗 */
const openForm = async (type: string, id?: number) => {
  dialogVisible.value = true
  dialogTitle.value = t('action.' + type)
  formType.value = type
  resetForm()
  // 修改时，设置数据
  if (id) {
    formLoading.value = true
    try {
      formData.value = await api.getDetail(id)
    } finally {
      formLoading.value = false
    }
  }
}

defineExpose({ openForm }) // 提供 open 方法，用于打开弹窗

/** 提交表单 */
const emit = defineEmits(['success']) // 定义 success 事件，用于操作成功后的回调

/** 提交表单 */
const submitForm = async () => {
  // 校验表单
  if (!formRef) return
  const valid = await formRef.value.validate()
  if (!valid) return
  for (const index in formData.value.deployTasks) {
    const deployTask = formData.value.deployTasks[index]
    const indexOne = parseInt(index) + 1
    if (!deployTask.projectId) {
      message.warning(`请检查部署配置，【第${indexOne}项部署项目】不能为空`)
      return
    }
    if (!deployTask.agentIds || !deployTask.agentIds.length || deployTask.agentIds.length == 0) {
      message.warning(`请检查部署配置，【第${indexOne}项部署机器】不能为空`)
      return
    }
    if (!deployTask.projectVersionId) {
      message.warning(`请检查部署配置，【第${indexOne}项部署版本】不能为空`)
      return
    }
  }

  // 提交请求
  formLoading.value = true
  try {
    const data = formData.value
    if (formType.value === 'create') {
      await api.create(data)
      message.success(t('common.createSuccess'))
    } else {
      await api.update(data)
      message.success(t('common.updateSuccess'))
    }
    dialogVisible.value = false
    emit('success')
  } finally {
    formLoading.value = false
  }
}

/** 重置表单 */
const resetForm = () => {
  formData.value = cloneDeep(defaultFormData)
  formRef.value?.resetFields()
}

/** 新增部署服务 */
const handleAddDeployTask = () => {
  formData.value.deployTasks.push({})
}

/** 移除部署服务 */
const handleRemoveDeployTask = (row: any) => {
  const index = formData.value.deployTasks.indexOf(row)
  if (index >= 0) {
    formData.value.deployTasks.splice(index, 1)
  }
}
</script>
