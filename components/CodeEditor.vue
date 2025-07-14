<script setup lang="ts">
import { ref, watch, onMounted } from "vue";
import { basicSetup } from "codemirror";
import { EditorView, keymap } from "@codemirror/view";
import { EditorState } from "@codemirror/state";
import { javascript } from "@codemirror/lang-javascript";
import { html } from "@codemirror/lang-html";
import { python } from "@codemirror/lang-python";
import { sql } from "@codemirror/lang-sql";
import { oneDark } from "@codemirror/theme-one-dark";
import { defaultKeymap, indentWithTab } from "@codemirror/commands";

const props = defineProps({
  modelValue: {
    type: String,
    default: "",
  },
  language: {
    type: String,
    default: "javascript",
    validator: (value: string) =>
      ["javascript", "html", "sql", "python"].includes(value),
  },
  darkMode: {
    type: Boolean,
    default: false,
  },
  readonly: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits(["update:modelValue"]);

const editorRef = ref<HTMLElement | null>(null);
const editorView = ref<EditorView | null>(null);

const getLanguageExtension = () => {
  switch (props.language) {
    case "javascript":
      return javascript();
    case "html":
      return html();
    case "python":
      return python();
    case "sql":
      return sql();
    default:
      return javascript();
  }
};

const createEditor = () => {
  if (!editorRef.value) return;

  const extensions = [
    basicSetup,
    keymap.of([...defaultKeymap, indentWithTab]),
    getLanguageExtension(),
    EditorView.lineWrapping,
    EditorView.updateListener.of((update) => {
      if (update.docChanged) {
        const doc = update.state.doc;
        const value = doc.toString();
        emit("update:modelValue", value);
      }
    }),
  ];

  if (props.darkMode) {
    extensions.push(oneDark);
  }

  if (props.readonly) {
    extensions.push(EditorState.readOnly.of(true));
  }

  editorView.value = new EditorView({
    state: EditorState.create({
      doc: props.modelValue,
      extensions,
    }),
    parent: editorRef.value,
  });
};

watch(
  () => props.modelValue,
  (newValue) => {
    if (editorView.value) {
      const currentValue = editorView.value.state.doc.toString();
      if (newValue !== currentValue) {
        editorView.value.dispatch({
          changes: {
            from: 0,
            to: currentValue.length,
            insert: newValue,
          },
        });
      }
    }
  }
);

watch(
  () => props.language,
  () => {
    if (editorView.value) {
      editorView.value.destroy();
      createEditor();
    }
  }
);

onMounted(() => {
  createEditor();
});

defineExpose({
  getValue: () => editorView.value?.state.doc.toString() || "",
  setValue: (value: string) => {
    if (editorView.value) {
      editorView.value.dispatch({
        changes: {
          from: 0,
          to: editorView.value.state.doc.length,
          insert: value,
        },
      });
    }
  },
});
</script>

<template>
  <div ref="editorRef" class="code-editor"></div>
</template>

<style>
.code-editor {
  border: 1px solid #ccc;
  border-radius: 4px;
  overflow: hidden;

}

.cm-editor {

  font-size: 14px;
  line-height: 1.5;
}

.cm-scroller {
  overflow: auto;
  font-family: "Fira Code", "Consolas", "Monaco", "Andale Mono", "Ubuntu Mono",
    monospace;
}

.cm-content {
  padding: 8px 0;
}

.cm-line {
  padding: 0 8px;
}
</style>
