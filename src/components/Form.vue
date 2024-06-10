<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2024-06-10 14:35
 * @desc       : 
-->
<script setup>
  import { ref, onMounted, watch } from 'vue';
  import { bitable, FieldType, DateFormatter } from '@lark-base-open/js-sdk';
  import { ElMessage, ElMessageBox } from 'element-plus';

  import MonacoEditor from './MonacoEditor.vue';
  import { useI18n } from 'vue-i18n';

  const { t } = useI18n();

  const base = bitable.base;

  const fieldOptions = ref();
  const fieldId = ref();

  const loading = ref(false);

  const tableId = ref('');
  const tableList = ref([]);

  const viewId = ref('');
  const viewList = ref([]);

  onMounted(async () => {
    // 数据表列表
    const tableMetaList = await base.getTableMetaList();
    tableList.value = tableMetaList.map((item) => ({ value: item.id, label: item.name }));

    // 获取字段列表
    const table = await base.getActiveTable();

    // 视图列表
    const viewMetaList = await table.getViewMetaList();
    viewList.value = viewMetaList.map((item) => ({ value: item.id, label: item.name }));

    const view = await table.getActiveView();

    viewId.value = view.id;
    tableId.value = table.id;

    const viewFieldList = await view.getFieldMetaList();
    fieldOptions.value = viewFieldList.map((item) => ({ value: item.id, label: item.name, type: item.type }));
  });

  watch(
    () => tableId.value,
    async (newTableId) => {
      const table = await bitable.base.getTable(newTableId);
      const viewMetaList = await table.getViewMetaList();
      viewList.value = viewMetaList.map((item) => ({ value: item.id, label: item.name }));
      viewId.value = viewList?.value[0]?.value;
    },
  );

  watch(
    () => viewId.value,
    async (newViewId) => {
      const table = await bitable.base.getTable(tableId.value);
      const view = await table.getViewById(newViewId);
      const viewFieldList = await view.getFieldMetaList();
      fieldOptions.value = viewFieldList.map((item) => ({ value: item.id, label: item.name, type: item.type }));
    },
  );

  async function confirm() {
    if (!appName.value) {
      ElMessage({
        type: 'error',
        message: t('Please enter the name of the program'),
      });
      return;
    }

    if (!fieldId.value) {
      ElMessage({
        type: 'error',
        message: t('p1'),
      });
      return;
    }

    if (!areaId.value) {
      ElMessage({
        type: 'error',
        message: t('p2'),
      });
      return;
    }

    generateBirthdayRow();
  }

  const areaId = ref();

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

  // const appName = ref(t('p3'));

  const appName = ref('hexToDecimal');

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
      >
        <el-icon style="margin-right: 2px"><Notebook /></el-icon>
        {{ $t('shi-yong-zhi-nan') }}</el-button
      >
      <el-button
        type="text"
        @click="
          goUrl('https://bcmcjimpjd.feishu.cn/base/I7AWbeSTLafqaJsTJ4BcmCF2nMg?table=tblTZGKJTASSgsyF&view=vewh3S13Xh')
        "
      >
        <el-icon style="margin-right: 2px"> <HomeFilled /></el-icon>
        {{ $t('cha-kan-yi-you-ying-yong') }}</el-button
      >
      <el-button
        type="text"
        @click="goUrl('https://bcmcjimpjd.feishu.cn/share/base/form/shrcnVjZKkMAQCKmU9eFuuUv1rh')"
      >
        <el-icon style="margin-right: 2px"><UploadFilled /></el-icon>
        {{ $t('ti-jiao-ying-yong') }}</el-button
      >
    </div>
    <div class="line">
      <el-button
        type="primary"
        @click="upload"
      >
        <el-icon style="margin-right: 2px"><Upload /></el-icon>
        {{ $t('dao-ru-ying-yong') }}</el-button
      >
      <el-button @click="download">
        <el-icon style="margin-right: 2px"><Download /></el-icon>
        {{ $t('dao-chu-ying-yong') }}</el-button
      >
    </div>
    <div class="line">
      <div class="title">{{ $t('Program Name') }}</div>
      <el-input
        v-model="appName"
        style="width: 60%"
        size="large"
        clearable
        :placeholder="$t('Please enter the name of the program')"
      />
    </div>

    <div class="line">
      <div class="title">
        <div>数据表</div>
      </div>
      <div>
        <el-select
          v-model="tableId"
          placeholder="请选择数据表"
          size="large"
          clearable
        >
          <el-option
            v-for="item in tableList"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>

    <div class="line">
      <div class="title">
        <div>视图</div>
      </div>
      <div>
        <el-select
          v-model="viewId"
          placeholder="请选择视图"
          size="large"
          clearable
        >
          <el-option
            v-for="item in viewList"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>
      </div>
    </div>

    <div class="line">
      <div class="title">
        <div>{{ $t('yuan-zi-duan') }}</div>
        <div class="box-item">
          <el-tooltip
            effect="dark"
            :content="$t('yuan-zi-duan-zhi-neng-wei-shu-zi-huo-wen-ben-lei-xing')"
            placement="right-start"
          >
            <el-icon><WarningFilled /></el-icon>
          </el-tooltip>
        </div>
      </div>
      <div>
        <el-select
          v-model="fieldId"
          :placeholder="$t('p1')"
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
      <div class="title">
        <div>{{ $t('mu-biao-zi-duan') }}</div>
        <div class="box-item">
          <el-tooltip
            effect="dark"
            :content="$t('mu-biao-zi-duan-zhi-neng-wei-wen-ben-lei-xing')"
            placement="right-start"
          >
            <el-icon><WarningFilled /></el-icon>
          </el-tooltip>
        </div>
      </div>
      <div>
        <el-select
          v-model="areaId"
          :placeholder="$t('p2')"
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

    <div class="area">
      {{ $t('dai-ma-bian-ji-qu-yu') }} <span class="tip">{{ $t('jin-zhi-chi-js-dai-ma') }}</span>
    </div>
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
    display: flex;
    font-size: 16px;
    margin-right: 10px;
    min-width: 100px;

    &::before {
      content: '*';
      color: red;
      margin-right: 5px;
    }

    .box-item {
      margin-left: 5px;
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
    margin-bottom: 20px;
  }

  .tip {
    color: red;
    font-size: 12px;
  }
</style>
