<template>
  <div class="op-line">
    <el-input style="width: 200px;" size="default" placeholder="部门名称" v-model="deptTitleKey"></el-input>
    <el-button size="default" plain style="vertical-align: middle; margin-left: 12px;" type="info" @click="expand">
      <SVGIcon style="width: 1em; height: 1em" name="Expand" /><span style="margin-left: 4px;">展开</span>
    </el-button>
    <el-button size="default" plain style="vertical-align: middle;" type="info" @click="shrink">
      <SVGIcon style="width: 1em; height: 1em" name="Shrink" /><span style="margin-left: 4px;">收缩</span>
    </el-button>
    <el-button size="default" plain style="vertical-align: middle;" type="primary" @click="addItem">
      <SVGIcon style="width: 1em; height: 1em" name="Plus"/><span style="margin-left: 4px;">新增</span>
    </el-button>
    <el-popconfirm
      title="确定删除?"
      confirmButtonText="确定"
      cancelButtonText="取消"
      @confirm="batchDelete"
    >
      <template #reference>
        <el-button size="default" plain style="vertical-align: middle;" type="danger" :disabled="currentSelectedList.length === 0">
          <SVGIcon style="width: 1em; height: 1em" name="Delete"/><span style="margin-left: 4px;">删除</span>
        </el-button>
      </template>
    </el-popconfirm>
    <el-button size="default" plain style="vertical-align: middle" type="warning" :icon="downloadIcon" @click="exportTable">导出</el-button>
  </div>

  <div class="data-table">
    <el-table
      v-loading="tableData.loading"
      ref="tableRef"
      :height="tableData.height"
      :data="filterDataList"
      style="width: 100%"
      row-key="id"
      :stripe="true"
      :tree-props="{ children: 'children' }"
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" align="center" header-align="center" />
      <el-table-column prop="simple_name" align="left" header-align="left" label="简称" width="300" />
      <el-table-column prop="id" align="center" header-align="center" label="#" width="60" />
      <el-table-column prop="ident" align="center" header-align="center" label="编号" width="100" />
      <el-table-column prop="order_no" align="center" header-align="center" label="排序" width="60" />
      <el-table-column prop="owner" align="center" header-align="center" label="领导" width="100">
        <template #default="scope">
          <el-tag>{{scope.row.owner > 0 ? scope.row.owner_info.nickname : '无'}}</el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="status" align="center" header-align="center" label="状态" width="80">
        <template #default="scope">
          <dict-tag ident="common_status" :val="scope.row.status"></dict-tag>
        </template>
      </el-table-column>
      <el-table-column label="操作">
        <template #default="scope">
          <el-button plain style="vertical-align: middle;" text @click="addItem(scope.row)" :icon="plusIcon">新增</el-button>
          <el-button plain style="vertical-align: middle;" text @click="addSubItem(scope.row)" :icon="plusIcon">子部门</el-button>
          <el-button plain style="vertical-align: middle;" text @click="editItem(scope.row)" :icon="editIcon">编辑</el-button>
          <el-popconfirm
            title="确定删除?"
            confirmButtonText="确定"
            cancelButtonText="取消"
            @confirm="delItem(scope.row)"
          >
            <template #reference>
              <el-button plain style="vertical-align: middle;" text :icon="deleteIcon">删除</el-button>
            </template>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>
  </div>

  <v-dialog
    v-model="dialogInfo.visible"
    :title="dialogInfo.title"
    width="600px"
    :draggable="true"
    :disable-footer="dialogInfo.loading"
    @cancel="dialogInfo.visible = false"
    @confirm="confirmDialog"
  >
    <el-form :model="dialogInfo.formData" size="default" :label-width="60" v-loading="dialogInfo.loading">
      <el-row v-if="dialogInfo.title === '编辑部门'">
        <el-form-item label="部门ID" style="width: 100%">
          <el-input disabled v-model="dialogInfo.formData.id"></el-input>
        </el-form-item>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item label="名称" style="width: 100%">
            <el-input v-model="dialogInfo.formData.title"></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="简称" style="width: 100%">
            <el-input v-model="dialogInfo.formData.simple_name"></el-input>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item label="标识" style="width: 100%">
            <el-input v-model="dialogInfo.formData.ident"></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="排序">
            <el-input-number :controls="false" v-model="dialogInfo.formData.order_no" style="width: 100%"></el-input-number>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item label="领导" style="width: 100%">
            <user-selector-input
              v-model="dialogInfo.formData.owner"
              value-key="id"
              :read-view-fn="k => dialogInfo.formData.owner_info"
              :multiple="false"
              placeholder="双击选择用户"
            ></user-selector-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="状态" style="width: 100%">
            <dict-input ident="common_status" v-model="dialogInfo.formData.status" ></dict-input>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-form-item label="描述" style="width: 100%">
          <el-input v-model="dialogInfo.formData.description"></el-input>
        </el-form-item>
      </el-row>

      <el-row>
        <el-col>
          <el-form-item label="父ID">
            <el-input-number :controls="false" v-model="dialogInfo.formData.pid" style="width: 100%"></el-input-number>
          </el-form-item>
        </el-col>
      </el-row>


    </el-form>
  </v-dialog>
</template>

<script lang="ts" setup>
import {onMounted, ref, computed, nextTick, inject, Ref, ComputedRef} from "vue";
import SVGIcon from "@/components/common/SVGIcon.vue";
import {
  ElTable,
  ElTableColumn,
  ElInput, ElTag, ElInputNumber,
  ElButton, ElRow, ElCol,
  ElPopconfirm, ElForm, ElFormItem, ElMessage
} from "element-plus";
import * as DeptApi from "@/api/sys/dept";
import {filterDataWithTitle} from "@/utils/common"
import {mainHeightKey, themeKey} from "@/config/app.keys";
import UserSelectorInput from "@/components/common/selector/user/UserSelectorInput.vue";
import DictInput from "@/components/dict/DictInput.vue";
import DictTag from "@/components/dict/DictTag.vue";
import {useIcon} from "@/components/common/util";
import VDialog from "@/components/dialog/VDialog.vue";

const deleteIcon = useIcon('Delete')
const editIcon = useIcon('Edit')
const plusIcon = useIcon('Plus')
const downloadIcon = useIcon('Download')

const tableRef = ref<InstanceType<typeof ElTable>>();

const theme = inject<Ref<ThemeConfig>>(themeKey)
const mainHeight = inject<ComputedRef<string>>(mainHeightKey)

const tableData = ref<TableData<DeptView>>({
  loading: false,
  height: computed<string>(() => `calc(${mainHeight.value} - ${theme.value.mainPadding + theme.value.mainPadding + 32 + 10}px)`),
  data: [],
})

const dialogInfo = ref<DialogInfo<AddDeptParam | UpdateDeptParam>>({
  visible: false,
  loading: false,
  title: '',
  formData: {
    id: 0,
    title: "",
    simple_name: "",
    description: "",
    ident: "",
    order_no: 0,
    pid: 0,
    owner: 0,
    status: false,
  }
})

const deptTitleKey = ref<string>("");

const formInit: AddDeptParam | UpdateDeptParam = {
  id: 0,
  title: "",
  simple_name: "",
  description: "",
  ident: "",
  order_no: 0,
  pid: 0,
  owner: 0,
  status: false,
};

const filterDataList = computed<DeptView[]>(() => {
  const result: DeptView[] = [];
  filterDataWithTitle(result, tableData.value.data, deptTitleKey.value, "simple_name", undefined);
  if (result.length > 0) nextTick(() => expandOrShrinkAll(result, true));
  return result;
});

function addItem(row: DeptView) {
  dialogInfo.value.title = "新增部门"
  if (row && !(row instanceof Event)) {
    dialogInfo.value.formData = {} as AddDeptParam
    Object.assign(dialogInfo.value.formData, row)
  } else {
    dialogInfo.value.formData = formInit
  }
  dialogInfo.value.visible = true;
}

function addSubItem(row: DeptView) {
  dialogInfo.value.title = "新增子部门"
  if (row && !(row instanceof Event)) {
    dialogInfo.value.formData = {} as AddDeptParam
    Object.assign(dialogInfo.value.formData, row)
  } else {
    dialogInfo.value.formData = formInit
  }
  dialogInfo.value.visible = true
}

function editItem(row: DeptView) {
  dialogInfo.value.title = "编辑部门";
  dialogInfo.value.formData = {} as UpdateDeptParam
  Object.assign(dialogInfo.value.formData, row)
  dialogInfo.value.visible = true

}


async function confirmDialog() {
  if (dialogInfo.value.title === '新增部门' || dialogInfo.value.title === '新增子部门') {
    await DeptApi.addDept(dialogInfo.value.formData as AddDeptParam)
    await reloadTableData()
    ElMessage.info("新增成功")
  } else if (dialogInfo.value.title === '编辑部门') {
    await DeptApi.updateDept(dialogInfo.value.formData as UpdateDeptParam)
    await reloadTableData()
    ElMessage.info("编辑成功")
  }
  dialogInfo.value.visible = false
}

function delItem(row: DeptView) {

}

function exportTable() {

}

function expand() {
  expandOrShrinkAll(filterDataList.value, true);
}

function shrink() {
  expandOrShrinkAll(filterDataList.value, false);
}

function expandOrShrinkAll(list: DeptView[], expanded: boolean) {
  for (let item of list) {
    if (item.children && item.children.length > 0) {
      tableRef.value.toggleRowExpansion(item, expanded);
      if (item.children && item.children.length > 0) {
        expandOrShrinkAll(item.children, expanded);
      }
    }
  }
}

const currentSelectedList = ref([]);

function handleSelectionChange(menus) {
  currentSelectedList.value = menus;
}

function batchDelete() {

}



async function reloadTableData() {
  tableData.value.loading = true;
  tableData.value.data = await DeptApi.findDept();
  tableData.value.loading = false;
}

onMounted(reloadTableData)
</script>

<style scoped>
.op-line {
  box-sizing: border-box;
  height: 32px;
}

.data-table {
  box-sizing: border-box;
  margin-top: 10px;
}

:deep(div.el-table__header-wrapper .el-table_1_column_1 div.cell) {
  display: none;
}
</style>
