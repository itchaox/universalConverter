<template>
  <div>
    <div
      ref="editorContainer"
      style="height: 50vh"
    ></div>
    <!-- <button @click="runCode('my name')">Run Code</button> -->
  </div>
</template>

<script setup>
  import { ref, onMounted } from 'vue';
  import * as monaco from 'monaco-editor';

  // 创建一个 ref 用于存储编辑器容器
  const editorContainer = ref(null);
  let editor = null;

  // 在组件挂载时初始化 Monaco Editor
  onMounted(() => {
    editor = monaco.editor.create(editorContainer.value, {
      value: `function helloWorld(data) {
    return data = data + '成功了！！！'
  }`,
      language: 'javascript',
    });
  });

  // 定义一个运行代码的函数
  const runCode = (data) => {
    const code = editor.getValue();
    try {
      // 创建一个新的函数，包含用户代码并返回其定义的函数
      const func = new Function(`${code}; return helloWorld;`);
      // 执行函数，得到用户定义的 helloWorld 函数
      const helloWorld = func();
      // 调用用户定义的 helloWorld 函数
      return helloWorld(data);
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
