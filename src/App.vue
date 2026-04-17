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

    <section style="height: 500px;"></section>

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
        聚焦后在移动端触发 touch 交互时会临时隐藏 caret；touchend 或 touchcancel 后，会根据光标位置恢复显示。若 caret 距离实际可视区域底部 150px 内，则会继续强制隐藏。
      </p>
    </section>

    <section class="panel panel-status">
      <h2>当前状态</h2>
      <p>{{ statusText }}</p>
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
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from "vue";

const initialText =
  "这是一段可编辑文字。聚焦后滚动页面，让编辑区进入顶部 mask 区域，观察 iOS 下 caret 的显示效果。";
const placeholder = "请输入内容，滚动试试遮罩与 caret";
const text = ref(initialText);
const editorRef = ref(null);
const maskRef = ref(null);
const isFocused = ref(false);
const shouldHideCaret = ref(false);
const isTouchActive = ref(false);
const isCaretNearViewportBottom = ref(false);
const bottomThreshold = 50;
const keyboardSafeGap = 24;
const focusScrollDelays = [0, 60, 140, 260, 420, 680, 900];

let caretUpdateFrame = null;
let focusScrollFrame = null;
let focusScrollTimers = [];
let focusScrollUntil = 0;

const statusText = computed(() => {
  if (isTouchActive.value) {
    return "当前 touch 进行中，caret 已隐藏";
  }

  if (isCaretNearViewportBottom.value) {
    return `caret 距离实际可视区域底部 ${bottomThreshold}px 内，已强制隐藏`;
  }

  return "当前无 touch，且 caret 未接近可视区域底部时，caret 正常显示";
});

function syncCaretVisibility() {
  shouldHideCaret.value =
    isFocused.value && (isTouchActive.value || isCaretNearViewportBottom.value);
}

function handleInput(event) {
  text.value = event.target.innerText;
  scheduleCaretPositionUpdate();
}

function getViewportBottom() {
  if (window.visualViewport) {
    return window.visualViewport.offsetTop + window.visualViewport.height;
  }

  return window.innerHeight;
}

function getViewportTop() {
  return window.visualViewport?.offsetTop ?? 0;
}

function getMaskBottom() {
  return maskRef.value?.getBoundingClientRect().bottom ?? 0;
}

function getCaretRange(selection) {
  const editor = editorRef.value;
  const focusNode = selection.focusNode;

  if (!editor || !focusNode || !editor.contains(focusNode)) {
    return null;
  }

  const range = document.createRange();
  range.setStart(focusNode, selection.focusOffset);
  range.collapse(true);
  return range;
}

function isUsefulRect(rect) {
  return rect && (rect.width > 0 || rect.height > 0);
}

function getCaretRectWithMarker(range, selection) {
  const marker = document.createElement("span");
  const originalRange = range.cloneRange();
  const markerRange = range.cloneRange();

  marker.textContent = "\u200b";
  marker.style.cssText =
    "display:inline-block;width:0;height:1em;overflow:hidden;line-height:1;vertical-align:baseline;";

  markerRange.insertNode(marker);
  const rect = marker.getBoundingClientRect();
  marker.remove();

  selection.removeAllRanges();
  selection.addRange(originalRange);

  return rect;
}

function getCaretRect() {
  const selection = window.getSelection();

  if (!selection || selection.rangeCount === 0) {
    return null;
  }

  const range = getCaretRange(selection);

  if (!range) {
    return null;
  }

  const rect = range.getBoundingClientRect();

  if (isUsefulRect(rect)) {
    return rect;
  }

  const clientRects = range.getClientRects();

  if (clientRects.length > 0 && isUsefulRect(clientRects[clientRects.length - 1])) {
    return clientRects[clientRects.length - 1];
  }

  return getCaretRectWithMarker(range, selection);
}

function updateCaretPositionState() {
  if (!isFocused.value) {
    isCaretNearViewportBottom.value = false;
    syncCaretVisibility();
    return;
  }

  const caretRect = getCaretRect();

  if (!caretRect || !isUsefulRect(caretRect)) {
    isCaretNearViewportBottom.value = false;
    syncCaretVisibility();
    return;
  }

  isCaretNearViewportBottom.value = getViewportBottom() - caretRect.bottom <= bottomThreshold;
  syncCaretVisibility();
}

function scrollPageBy(delta) {
  if (Math.abs(delta) < 1) {
    return;
  }

  window.scrollTo({
    top: window.scrollY + delta,
    behavior: "auto",
  });
}

function adjustEditorIntoVisualViewport() {
  const editor = editorRef.value;

  if (!isFocused.value || !editor) {
    return;
  }

  const rect = editor.getBoundingClientRect();

  if (!isUsefulRect(rect)) {
    return;
  }

  const safeTop = Math.max(getViewportTop(), getMaskBottom()) + keyboardSafeGap;
  const safeBottom = getViewportBottom() - keyboardSafeGap;
  const safeHeight = safeBottom - safeTop;

  if (safeHeight <= 0) {
    return;
  }

  if (rect.height > safeHeight && rect.top !== safeTop) {
    scrollPageBy(rect.top - safeTop);
    return;
  }

  if (rect.bottom > safeBottom) {
    scrollPageBy(rect.bottom - safeBottom);
    return;
  }

  if (rect.top < safeTop) {
    scrollPageBy(rect.top - safeTop);
  }
}

function scheduleFocusScroll() {
  if (focusScrollFrame !== null) {
    window.cancelAnimationFrame(focusScrollFrame);
  }

  focusScrollFrame = window.requestAnimationFrame(() => {
    focusScrollFrame = null;
    adjustEditorIntoVisualViewport();
  });
}

function clearFocusScrollTimers() {
  focusScrollTimers.forEach((timer) => window.clearTimeout(timer));
  focusScrollTimers = [];
}

function startFocusScrollWindow() {
  clearFocusScrollTimers();
  focusScrollUntil = Date.now() + focusScrollDelays[focusScrollDelays.length - 1];
  focusScrollTimers = focusScrollDelays.map((delay) =>
    window.setTimeout(scheduleFocusScroll, delay),
  );
}

function handleViewportChange() {
  scheduleCaretPositionUpdate();

  if (isFocused.value && Date.now() <= focusScrollUntil) {
    scheduleFocusScroll();
  }
}

function focusEditableElement(editor) {
  try {
    editor.focus({ preventScroll: true });
  } catch {
    editor.focus();
  }
}

function scheduleCaretPositionUpdate() {
  if (caretUpdateFrame !== null) {
    window.cancelAnimationFrame(caretUpdateFrame);
  }

  caretUpdateFrame = window.requestAnimationFrame(() => {
    caretUpdateFrame = null;
    updateCaretPositionState();
  });
}

function startInteraction() {
  isTouchActive.value = true;
  syncCaretVisibility();
}

function endInteraction() {
  isTouchActive.value = false;
  scheduleCaretPositionUpdate();
}

function handleFocus() {
  isFocused.value = true;
  scheduleCaretPositionUpdate();
  startFocusScrollWindow();
}

function handleBlur() {
  isFocused.value = false;
  isCaretNearViewportBottom.value = false;
  shouldHideCaret.value = false;
  focusScrollUntil = 0;
  clearFocusScrollTimers();
}

async function focusEditor() {
  await nextTick();
  const editor = editorRef.value;

  if (editor) {
    focusEditableElement(editor);
    scheduleCaretPositionUpdate();
    startFocusScrollWindow();
  }
}

async function resetText() {
  text.value = initialText;
  await nextTick();

  if (editorRef.value) {
    editorRef.value.innerText = initialText;
    focusEditableElement(editorRef.value);
    scheduleCaretPositionUpdate();
    startFocusScrollWindow();
  }
}

onMounted(() => {
  document.addEventListener("selectionchange", scheduleCaretPositionUpdate);
  window.addEventListener("touchstart", startInteraction, { passive: true });
  window.addEventListener("touchmove", startInteraction, { passive: true });
  window.addEventListener("touchend", endInteraction, { passive: true });
  window.addEventListener("touchcancel", endInteraction, { passive: true });
  window.visualViewport?.addEventListener("resize", handleViewportChange, { passive: true });
  window.visualViewport?.addEventListener("scroll", handleViewportChange, { passive: true });
  focusEditor();
});

onBeforeUnmount(() => {
  if (caretUpdateFrame !== null) {
    window.cancelAnimationFrame(caretUpdateFrame);
  }

  if (focusScrollFrame !== null) {
    window.cancelAnimationFrame(focusScrollFrame);
  }

  clearFocusScrollTimers();
  document.removeEventListener("selectionchange", scheduleCaretPositionUpdate);
  window.removeEventListener("touchstart", startInteraction);
  window.removeEventListener("touchmove", startInteraction);
  window.removeEventListener("touchend", endInteraction);
  window.removeEventListener("touchcancel", endInteraction);
  window.visualViewport?.removeEventListener("resize", handleViewportChange);
  window.visualViewport?.removeEventListener("scroll", handleViewportChange);
});
</script>
