<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2024-06-08 12:59
 * @desc       : 
-->
<script setup>
  import { ref, onMounted } from 'vue';
  import { bitable, FieldType, DateFormatter } from '@lark-base-open/js-sdk';
  import { ElMessage, ElMessageBox } from 'element-plus';

  import MonacoEditor from './MonacoEditor.vue';
  import { useI18n } from 'vue-i18n';

  const { t } = useI18n();

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  const loading = ref(false);

  onMounted(async () => {
    const table = await base.getActiveTable();
    const tableMetaList = await table.getFieldMetaList();
    fieldOptions.value = tableMetaList.map((item) => ({ value: item.id, label: item.name, type: item.type }));
  });

  async function confirm() {
    if (!appName.value) {
      ElMessage({
        type: 'error',
        message: '请输入应用名字',
      });
      return;
    }

    if (!fieldId.value) {
      ElMessage({
        type: 'error',
        message: '请选择原始列',
      });
      return;
    }

    if (!areaId.value) {
      ElMessage({
        type: 'error',
        message: '请选择目标列',
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
      message: t('Data processing completed'),
      type: 'success',
      duration: 500,
    });
  }

  const MonacoEditorRef = ref();

  const appName = ref('万能转换器');

  const download = () => {
    const obj = {
      appName: appName.value,
      codeValue: MonacoEditorRef.value?.getEditorValue(),
    };

    // 将数据转换为 JSON 格式
    let jsonData = JSON.stringify(obj, null, 2);

    // 创建一个 Blob 对象
    let blob = new Blob([jsonData], { type: 'application/json' });

    // 创建一个链接
    let url = URL.createObjectURL(blob);

    // 创建一个链接元素
    let a = document.createElement('a');
    a.href = url;
    a.download = `${appName.value}`; // 文件名

    // 模拟点击下载链接
    a.click();

    // 释放资源
    URL.revokeObjectURL(url);
  };

  const isUpload = ref(false);
  function upload() {
    isUpload.value = true;
  }

  function cancelImport() {
    isUpload.value = false;
  }

  // 上传之前的处理
  function beforeUpload(file) {
    // 检查文件类型
    const isJSON = file.type === 'application/json';
    if (!isJSON) {
      ElMessage.error(t('Only supports uploading JSON files'));
    }
    return isJSON;
  }

  const codeValue = ref();

  const temAppName = ref();
  const temCodeValue = ref();

  // 上传成功后的处理
  function handleUploadSuccess(response, file) {
    // 获取上传的文件名
    const fileName = file[0].name;

    // 截取以 .json 结尾的前面的数据
    const lastIndex = fileName.lastIndexOf('.json');
    const truncatedFileName = fileName.slice(0, lastIndex);

    // 获取上传的 JSON 文件
    const reader = new FileReader();

    reader.onload = (e) => {
      // 将文件内容解析为 JavaScript 对象
      const res = JSON.parse(e.target.result);
      temCodeValue.value = res.codeValue;

      temAppName.value = truncatedFileName;
      ElMessage.success(t('File uploaded successfully'));
    };

    reader.readAsText(file[0].raw);
  }

  const uploadRef = ref();
  async function confirmImport() {
    appName.value = temAppName.value;
    codeValue.value = temCodeValue.value;
    isUpload.value = false;

    ElMessage({
      type: 'success',
      message: `${t('Added successfully')}`,
      duration: 1500,
      showClose: true,
    });

    uploadRef.value.clearFiles();
  }

  const goUrl = (url) => {
    window.open(url);
  };
</script>

<template>
  <div
    v-loading="loading"
    :element-loading-text="$t('loading')"
  >
    <div class="line">
      <el-button
        type="text"
        @click="goUrl('https://bcmcjimpjd.feishu.cn/base/I7AWbeSTLafqaJsTJ4BcmCF2nMg?table=ldxyob7oZYiCcGzh')"
        >使用指南</el-button
      >
      <el-button
        type="text"
        @click="
          goUrl('https://bcmcjimpjd.feishu.cn/base/I7AWbeSTLafqaJsTJ4BcmCF2nMg?table=tblTZGKJTASSgsyF&view=vewh3S13Xh')
        "
        >查看已有应用</el-button
      >
      <el-button
        type="text"
        @click="goUrl('https://bcmcjimpjd.feishu.cn/share/base/form/shrcnVjZKkMAQCKmU9eFuuUv1rh')"
        >提交应用</el-button
      >
    </div>
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
            v-for="item in fieldOptions?.filter(
              (item) =>
                item.value !== areaId &&
                [1, 2, 13, 15, 22, 1005, 99001, 99002, 99003, 99004, 99005].includes(item.type),
            )"
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
            v-for="item in fieldOptions?.filter((item) => item.value !== fieldId && [1, 15, 22].includes(item.type))"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>

    <h2>代码编辑区域</h2>
    <div>* 仅支持 js 代码</div>
    <MonacoEditor
      ref="MonacoEditorRef"
      class="editor"
      :codeValue="codeValue"
    />

    <!-- 导入方案 -->
    <el-dialog
      v-model="isUpload"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      @close="cancelImport"
      :title="$t('upload Programs')"
      width="75%"
    >
      <div class="addView">
        <el-upload
          ref="uploadRef"
          drag
          :http-request="() => {}"
          :on-change="handleUploadSuccess"
          :before-upload="beforeUpload"
          :limit="1"
        >
          <el-icon class="el-icon--upload"><upload-filled /></el-icon>
          <div class="el-upload__text">
            <em>{{ $t('drag') }}</em> {{ $t('or') }} <em>{{ $t('Click to upload a file') }}</em>
          </div>
        </el-upload>

        <div>
          <el-button
            type="primary"
            @click="confirmImport"
            >{{ $t('confirm') }}</el-button
          >

          <el-button @click="cancelImport">{{ $t('cancel') }}</el-button>
        </div>
      </div>
    </el-dialog>

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
    >
      <el-icon size="22"><CaretRight /></el-icon>
      {{ $t('run') }}</el-button
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
      content: '*';
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
