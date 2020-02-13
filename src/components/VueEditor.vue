<template>
  <div class="quillWrapper">
    <slot name="toolbar"></slot>
    <div :id="id" ref="quillContainer"></div>
    <input
      v-if="useCustomImageHandler"
      id="file-upload"
      ref="fileInput"
      type="file"
      accept="image/*"
      style="display:none;"
      @change="emitImageInfo($event)"
    />
  </div>
</template>

<script>
import Quill from "quill";
import ImageResize from "quill-image-resize";
import QuillImageDropAndPaste from "quill-image-drop-and-paste";
import hljs from "highlight.js";
import defaultToolbar from "@/helpers/default-toolbar";
import oldApi from "@/helpers/old-api";
import mergeDeep from "@/helpers/merge-deep";
import MarkdownShortcuts from "@/helpers/markdown-shortcuts";

export default {
  name: "VueEditor",
  mixins: [oldApi],
  props: {
    id: {
      type: String,
      default: "quill-container"
    },
    placeholder: {
      type: String,
      default: ""
    },
    value: {
      type: String,
      default: ""
    },
    disabled: {
      type: Boolean
    },
    editorToolbar: {
      type: Array,
      default: () => []
    },
    editorOptions: {
      type: Object,
      required: false,
      default: () => ({})
    },
    useCustomImageHandler: {
      type: Boolean,
      default: false
    },
    useMarkdownShortcuts: {
      type: Boolean,
      default: false
    },
    imageUploadUrl: {
      type: String,
      default: ""
    }
  },

  data: () => ({
    quill: null
  }),

  watch: {
    value(val) {
      if (val != this.quill.root.innerHTML && !this.quill.hasFocus()) {
        this.quill.root.innerHTML = val;
      }
    },
    disabled(status) {
      this.quill.enable(!status);
    }
  },

  mounted() {
    this.registerCustomModules(Quill);
    this.registerPrototypes();
    this.initializeEditor();
  },

  beforeDestroy() {
    this.quill = null;
    delete this.quill;
  },

  methods: {
    initializeEditor() {
      this.setupQuillEditor();
      this.checkForCustomImageHandler();
      this.handleInitialContent();
      this.registerEditorEventListeners();
      this.$emit("ready", this.quill);
    },

    setupQuillEditor() {
      let editorConfig = {
        debug: false,
        modules: this.setModules(),
        theme: "snow",
        placeholder: this.placeholder ? this.placeholder : "",
        readOnly: this.disabled ? this.disabled : false
      };

      this.prepareEditorConfig(editorConfig);

      Quill.register("modules/imageResize", ImageResize, true);
      Quill.register("modules/imageDropAndPaste", QuillImageDropAndPaste, true);

      this.quill = new Quill(this.$refs.quillContainer, editorConfig);
    },

    setModules() {
      let modules = {
        syntax: {
          highlight: text => hljs.highlightAuto(text).value
        },
        toolbar: this.editorToolbar.length
          ? this.editorToolbar
          : defaultToolbar,
        imageDropAndPaste: {
          handler: this.imageHandler
        },
        imageResize: {
          moduels: ["Resize", "DisplaySize", "Toolbar"]
        }
      };
      if (this.useMarkdownShortcuts) {
        Quill.register("modules/markdownShortcuts", MarkdownShortcuts, true);
        modules["markdownShortcuts"] = {};
      }
      return modules;
    },

    prepareEditorConfig(editorConfig) {
      if (
        Object.keys(this.editorOptions).length > 0 &&
        this.editorOptions.constructor === Object
      ) {
        if (
          this.editorOptions.modules &&
          typeof this.editorOptions.modules.toolbar !== "undefined"
        ) {
          // We don't want to merge default toolbar with provided toolbar.
          delete editorConfig.modules.toolbar;
        }

        mergeDeep(editorConfig, this.editorOptions);
      }
    },

    registerPrototypes() {
      Quill.prototype.getHTML = function() {
        return this.container.querySelector(".ql-editor").innerHTML;
      };
      Quill.prototype.getWordCount = function() {
        return this.container.querySelector(".ql-editor").innerText.length;
      };
    },

    registerEditorEventListeners() {
      this.quill.on("text-change", this.handleTextChange);
      this.quill.on("selection-change", this.handleSelectionChange);
      this.listenForEditorEvent("text-change");
      this.listenForEditorEvent("selection-change");
      this.listenForEditorEvent("editor-change");
    },

    listenForEditorEvent(type) {
      this.quill.on(type, (...args) => {
        this.$emit(type, ...args);
      });
    },

    handleInitialContent() {
      if (this.value) this.quill.root.innerHTML = this.value; // Set initial editor content
    },

    handleSelectionChange(range, oldRange) {
      if (!range && oldRange) this.$emit("blur", this.quill);
      else if (range && !oldRange) this.$emit("focus", this.quill);
    },

    handleTextChange(delta, oldContents) {
      let editorContent =
        this.quill.getHTML() === "<p><br></p>" ? "" : this.quill.getHTML();
      this.$emit("input", editorContent);

      if (this.useCustomImageHandler)
        this.handleImageRemoved(delta, oldContents);
    },

    handleImageRemoved(delta, oldContents) {
      const currrentContents = this.quill.getContents();
      const deletedContents = currrentContents.diff(oldContents);
      const operations = deletedContents.ops;

      operations.map(operation => {
        if (operation.insert && operation.insert.hasOwnProperty("image")) {
          const { image } = operation.insert;
          this.$emit("image-removed", image);
        }
      });
    },
    checkForCustomImageHandler() {
      this.useCustomImageHandler === true ? this.setupCustomImageHandler() : "";
    },

    setupCustomImageHandler() {
      let toolbar = this.quill.getModule("toolbar");
      toolbar.addHandler("image", this.customImageHandler);
    },

    //customImageHandler(image, callback) {
    customImageHandler() {
      this.$refs.fileInput.click();
    },

    emitImageInfo($event) {
      const resetUploader = function() {
        var uploader = document.getElementById("file-upload");
        uploader.value = "";
      };
      let file = $event.target.files[0];
      let Editor = this.quill;
      let cursorLocation =
        (this.quill.getSelection() || {}).index || this.quill.getLength();
      this.$emit("image-added", file, Editor, cursorLocation, resetUploader);
    },
    imageHandler(imageDataUrl, type) {
      // give a default mime type if the type was null
      if (!type) type = "image/png";

      //base64 to blob
      let binStr = atob(imageDataUrl.replace(/^data:image\/\w+;base64,/, ""));
      let arr = new Uint8Array(binStr.length);
      for (let i = 0; i < binStr.length; i++) {
        arr[i] = binStr.charCodeAt(i);
      }

      let filename = [
        "editor-img",
        "-",
        Math.floor(Math.random() * 1e12),
        ".",
        type.match(/^image\/(\w+)$/i)[1]
      ].join("");

      let file = new File([arr], filename, { type: type });

      let Editor = this.quill;
      let cursorLocation =
        (this.quill.getSelection() || {}).index || this.quill.getLength();
      this.$emit("image-added", file, Editor, cursorLocation);
    }
  }
};
</script>

<style src="quill/dist/quill.snow.css"></style>
<style src="../assets/vue2-editor.scss" lang="scss"></style>
<style src="highlight.js/styles/monokai-sublime.css"></style>
