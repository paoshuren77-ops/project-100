<template>
  <main class="page">
    <div ref="maskRef" class="mask">
      <div class="mask__label">Fixed mask</div>
    </div>

    <section class="hero">
      <p class="eyebrow">Vue 3 + contenteditable</p>
      <h1>iOS 遮罩与光标测试页</h1>
      <p class="intro">
        这个示例使用 Vue 3 渲染一个 <code>contenteditable</code> 区域，并模拟页面滚动时穿过
        固定遮罩的场景。
      </p>
    </section>

    <section class="panel">
      <label class="label" for="editor">可编辑区域</label>
      <div
        id="editor"
        ref="editorRef"
        :class="['editor', { 'editor-caret-hidden': shouldHideCaret }]"
        contenteditable="true"
        spellcheck="false"
        autocapitalize="off"
        autocomplete="off"
        role="textbox"
        aria-multiline="true"
        :data-placeholder="placeholder"
        @focus="handleFocus"
        @blur="handleBlur"
        @input="handleInput"
      >{{ initialText }}</div>

      <div class="toolbar">
        <button type="button" class="button" @click="focusEditor">聚焦编辑区</button>
        <button type="button" class="button button-secondary" @click="resetText">
          重置示例文本
        </button>
      </div>

      <p class="hint">
        聚焦后向上滚动，让编辑区进入顶部遮罩区域。进入遮罩区域时会隐藏 caret，模拟“被遮住”的视觉效果，同时尽量保持键盘不收起。
      </p>
    </section>

    <section class="panel panel-status">
      <h2>当前状态</h2>
      <p>{{ shouldHideCaret ? "编辑区进入遮罩区域，caret 已隐藏" : "编辑区未进入遮罩区域，caret 正常显示" }}</p>
    </section>

    <section class="panel">
      <h2>当前内容</h2>
      <pre class="preview">{{ text }}</pre>
    </section>

    <section class="panel">
      <h2>排查要点</h2>
      <ul class="checklist">
        <li>保留 <code>user-select: text</code> 与 <code>-webkit-touch-callout: default</code></li>
        <li>不要给编辑区或父元素设置 <code>touch-action: none</code></li>
        <li>不要在 <code>touchstart</code>、<code>pointerdown</code> 等事件里调用 <code>preventDefault()</code></li>
        <li>iOS 下无法稳定依赖 <code>fixed + z-index</code> 压住原生 caret，只能用隐藏 caret 的方式近似处理</li>
      </ul>
    </section>

    <div class="page-spacer" aria-hidden="true"></div>
  </main>
</template>

<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref } from "vue";

const initialText =
  "这是一段可编辑文字。聚焦后滚动页面，让编辑区进入顶部 mask 区域，观察 iOS 下 caret 的显示效果。";
const placeholder = "请输入内容，滚动试试遮罩与 caret";
const text = ref(initialText);
const editorRef = ref(null);
const maskRef = ref(null);
const isFocused = ref(false);
const shouldHideCaret = ref(false);

function handleInput(event) {
  text.value = event.target.innerText;
}

function updateMaskState() {
  const editor = editorRef.value;
  const mask = maskRef.value;

  if (!editor || !mask || !isFocused.value) {
    shouldHideCaret.value = false;
    return;
  }

  const editorRect = editor.getBoundingClientRect();
  const maskRect = mask.getBoundingClientRect();
  shouldHideCaret.value = editorRect.top < maskRect.bottom && editorRect.bottom > maskRect.top;
}

function handleFocus() {
  isFocused.value = true;
  updateMaskState();
}

function handleBlur() {
  isFocused.value = false;
  shouldHideCaret.value = false;
}

async function focusEditor() {
  await nextTick();
  editorRef.value?.focus();
  updateMaskState();
}

async function resetText() {
  text.value = initialText;
  await nextTick();

  if (editorRef.value) {
    editorRef.value.innerText = initialText;
    editorRef.value.focus();
    updateMaskState();
  }
}

onMounted(() => {
  window.addEventListener("scroll", updateMaskState, { passive: true });
  window.addEventListener("resize", updateMaskState, { passive: true });
  focusEditor();
});

onBeforeUnmount(() => {
  window.removeEventListener("scroll", updateMaskState);
  window.removeEventListener("resize", updateMaskState);
});
</script>
