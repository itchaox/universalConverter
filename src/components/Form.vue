<!--
 * @Version    : v1.00
 * @Author     : wangchao
 * @Date       : 2024-06-07 16:17
 * @LastAuthor : wangchao
 * @LastTime   : 2024-06-07 17:22
 * @desc       : 
-->
<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2024-04-19 14:08
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from "vue";
  import { bitable, FieldType, DateFormatter } from "@lark-base-open/js-sdk";
  import { ElMessage, ElMessageBox } from "element-plus";
  import find from "../utils";

  import MonacoEditor from "./MonacoEditor.vue";
  import { useI18n } from "vue-i18n";

  const { t } = useI18n();

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  const loading = ref(false);

  onMounted(async () => {
    const table = await base.getActiveTable();
    const tableMetaList = await table.getFieldMetaList();
    fieldOptions.value = tableMetaList.map((item) => ({ value: item.id, label: item.name }));
  });

  async function confirm() {
    if (!appName.value) {
      ElMessage({
        type: "error",
        message: "请输入应用名字",
      });
      return;
    }

    if (!fieldId.value) {
      ElMessage({
        type: "error",
        message: "请选择原始列",
      });
      return;
    }

    if (!areaId.value) {
      ElMessage({
        type: "error",
        message: "请选择目标列",
      });
      return;
    }

    generateBirthdayRow();
  }

  // 所属地列
  const areaId = ref();

  /**
   * @desc  : 生成手机号码所属地列
   */
  async function generateBirthdayRow() {
    loading.value = true;

    const table = await base.getActiveTable();
    const recordList = await table.getRecordList();
    const field = await table.getField(areaId.value); // 选择某个多行文本字段
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    let _list = [];
    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;

      if (!val) continue;
      const _value = val[0]?.text;

      // 根据手机号码获取手机号码所属地
      _list.push({
        recordId: recordIds[index],
        fields: {
          [field.id]: MonacoEditorRef.value.runCode(_value),
        },
      });
    }

    await table.setRecords(_list);

    loading.value = false;
    ElMessage({
      message: t("Data processing completed"),
      type: "success",
    });
  }

  const MonacoEditorRef = ref();

  const appName = ref("万能转换器");

  const upload = () => {};

  const download = () => {
    console.log("MonacoEditorRef.value?.editor", MonacoEditorRef.value?.getEditorValue());

    const obj = {
      appName: app.value,
      codeValue: MonacoEditorRef.value?.editor.getValue(),
    };

    // 将数据转换为 JSON 格式
    let jsonData = JSON.stringify(obj, null, 2);

    // 创建一个 Blob 对象
    let blob = new Blob([jsonData], { type: "application/json" });

    // 创建一个链接
    let url = URL.createObjectURL(blob);

    // 创建一个链接元素
    let a = document.createElement("a");
    a.href = url;
    a.download = `${appName.value}`; // 文件名

    // 模拟点击下载链接
    a.click();

    // 释放资源
    URL.revokeObjectURL(url);
  };
</script>

<template>
  <div
    v-loading="loading"
    :element-loading-text="$t('loading')"
  >
    <div class="line">
      <el-button
        type="primary"
        @click="upload"
        >导入应用</el-button
      >
      <el-button @click="download">导出应用</el-button>
    </div>
    <div class="line">
      <div class="title">应用名字：</div>
      <el-input
        v-model="appName"
        style="width: 40%"
        size="large"
        clearable
      />
    </div>
    <div class="line">
      <div class="title">原始列：</div>
      <div>
        <el-select
          v-model="fieldId"
          placeholder="请选择原始列"
          size="large"
          clearable
        >
          <el-option
            v-for="item in fieldOptions?.filter((item) => item.value !== areaId)"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>

    <div class="line">
      <div class="title top">目标列：</div>
      <div>
        <el-select
          v-model="areaId"
          placeholder="请选择目标列"
          size="large"
          clearable
        >
          <el-option
            v-for="item in fieldOptions?.filter((item) => item.value !== fieldId)"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>

    <h2>代码编辑区域</h2>
    <MonacoEditor
      ref="MonacoEditorRef"
      class="editor"
    />

    <!-- 
    <div>
      <span>是否保存在本地</span>
      <el-switch>是否保存在本地</el-switch>
    </div> -->

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
    >
      <el-icon size="22"><CaretRight /></el-icon>
      {{ $t("run") }}</el-button
    >
  </div>
</template>

<style scoped>
  .title {
    font-size: 16px;
    font-weight: 700;
    margin-bottom: 14px;
    width: 100px;

    &::before {
      content: "*";
      color: red;
      margin-right: 5px;
    }
  }

  .editor {
    margin: 20px 0;
    border: 1px solid #303030;
    padding: 2px 0;
  }

  .top {
    margin-top: 20px;
  }

  .line {
    display: flex;
    align-items: center;
    margin-bottom: 14px;
  }
</style>
