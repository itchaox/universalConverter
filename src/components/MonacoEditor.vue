<template>
  <div>
    <div
      ref="editorContainer"
      style="height: 35vh"
    ></div>
    <!-- <button @click="runCode('my name')">Run Code</button> -->
  </div>
</template>

<script setup>
  import { ref, onMounted, watch } from 'vue';
  import * as monaco from 'monaco-editor';
  import { bitable } from '@lark-base-open/js-sdk';
  const theme = ref('');

  onMounted(async () => {
    theme.value = await bitable.bridge.getTheme();
  });

  bitable.bridge.onThemeChange((event) => {
    theme.value = event.data.theme;
  });

  // 定义 props
  const props = defineProps({
    codeValue: String,
  });

  // 创建一个 ref 用于存储编辑器容器
  const editorContainer = ref(null);
  let editor = null;

  const defaultValue = `function helloWorld(data) {
  // Some notes
    const res = data + 'some message!'
    return res
  }`;

  // 在组件挂载时初始化 Monaco Editor
  onMounted(() => {
    editor = monaco.editor.create(editorContainer.value, {
      value: defaultValue,
      language: 'javascript',
    });
  });

  // 监听 props.codeValue 的变化，并更新编辑器的值
  watch(
    () => props.codeValue,
    (newValue) => {
      if (editor) {
        const currentValue = editor.getValue();
        if (currentValue !== newValue) {
          editor.setValue(newValue);
        }
      }
    },
  );

  watch(
    () => theme.value,
    (newValue) => {
      if (editor) {
        console.log(newValue);
        if (newValue === 'DARK') {
          monaco.editor.setTheme('vs-dark');
        } else {
          monaco.editor.setTheme('vs');
        }
      }
    },
  );

  // 定义一个运行代码的函数
  const runCode = (data) => {
    const code = editor.getValue();
    try {
      // 使用正则表达式提取函数名
      const functionNameMatch = code.match(/function\s+([a-zA-Z_$][0-9a-zA-Z_$]*)\s*\(/);
      if (!functionNameMatch) {
        throw new Error('No function found in the code');
      }

      const functionName = functionNameMatch[1];
      const func = new Function(`${code}; return ${functionName};`);

      // 执行函数，得到用户定义的目标函数
      const userFunction = func();

      // 调用用户定义的目标函数, 全部转换成 string
      return userFunction(data) + '';
    } catch (e) {
      console.error(e);
    }
  };

  // 定义一个获取 editor 值的方法
  const getEditorValue = () => {
    return editor.getValue();
  };

  defineExpose({ runCode, getEditorValue });
</script>

<style>
  #editorContainer {
    border: 1px solid #ccc;
  }
</style>
