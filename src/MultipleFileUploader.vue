<template lang="pug">
.uploadBox
  h3 {{ headerMessage }}
  form(role="form", enctype="multipart/form-data", @submit.prevent="onSubmit")
    .uploadBoxMain(v-if="!itemsAdded")
      .form-group
        .dropArea(
          @ondragover="onChange",
          :class="dragging ? 'dropAreaDragging' : ''",
          @dragenter="dragging = true",
          @dragend="dragging = false",
          @dragleave="dragging = false"
        )
          h3 {{ dropAreaPrimaryMessage }}
          input#items(
            type="file",
            name="items[]",
            required="",
            multiple="",
            @change="onChange"
          )
          p.help-block {{ dropAreaSecondaryMessage }}
    .uploadBoxMain(v-else="")
      p
        strong {{ fileNameMessage }}
      ol
        li(v-for="name in itemsNames") {{ name }}
      p
        strong {{ fileSizeMessage }}
      ol
        li(v-for="size in itemsSizes") {{ size }}
      p
        strong {{ totalFileMessage }}
        |
        | {{ itemsAdded }}
      p
        strong {{ totalUploadSizeMessage }}
        |
        | {{ itemsTotalSize }}
      button(@click="removeItems") {{ removeFileMessage }}
      .loader(v-if="isLoaderVisible")
        .loaderImg
    div
      button.btn.btn-primary.btn-black.btn-round(
        type="submit",
        :disabled="itemsAdded < minItems || itemsAdded > maxItems"
      )
        | {{ uploadButtonMessage }}
      button.btn.btn-default.btn-round(type="button", @click="removeItems") {{ cancelButtonMessage }}
    br
    .successMsg(v-if="successMsg !== ''") {{ successMsg }}
    .errorMsg(v-if="errorMsg !== ''")
      | {{ fileUploadErrorMessage }}:
      br
      | {{ errorMsg }}
      br
      | {{ retryErrorMessage }}
    .errorMsg(v-if="itemsAdded && itemsAdded < minItems")
      | {{ minFilesErrorMessage }}: {{ minItems }}.
      br
      | {{ retryErrorMessage }}
    .errorMsg(v-if="itemsAdded && itemsAdded > maxItems")
      | {{ maxFilesErrorMessage }}: {{ maxItems }}.
      br
      | {{ retryErrorMessage }}
</template>

<script>
require('es6-promise').polyfill();
import axios from 'axios';

component: { axios }
export default {
  props: {
    postURL: {
      type: String,
      required: true
    },
    minItems: {
      type: Number,
      default: 1
    },
    maxItems: {
      type: Number,
      default: 30
    },
    method: {
      type: String,
      default: 'post'
    },
    postMeta: {
      type: [String, Array, Object],
      default: ''
    },
    postData: {
      type: [Object],
      default: () => { }
    },
    postHeader: {
      type: [Object],
      default: () => { }
    },
    successMessagePath: {
      type: String,
      required: true
    },
    errorMessagePath: {
      type: String,
      required: true
    },
    headerMessage: {
      type: String,
      default: 'Add files'
    },
    dropAreaPrimaryMessage: {
      type: String,
      default: 'Drop multiple files here'
    },
    dropAreaSecondaryMessage: {
      type: String,
      default: 'or click to upload'
    },
    fileNameMessage: {
      type: String,
      default: 'Names'
    },
    fileSizeMessage: {
      type: String,
      default: 'Sizes'
    },
    totalFileMessage: {
      type: String,
      default: 'Total files:'
    },
    totalUploadSizeMessage: {
      type: String,
      default: 'Total upload size:'
    },
    removeFileMessage: {
      type: String,
      default: 'Remove files'
    },
    uploadButtonMessage: {
      type: String,
      default: 'Upload'
    },
    cancelButtonMessage: {
      type: String,
      default: 'Cancel'
    },
    fileUploadErrorMessage: {
      type: String,
      default: 'An error has occurred'
    },
    minFilesErrorMessage: {
      type: String,
      default: 'Minimum files that need to be added to uploader'
    },
    maxFilesErrorMessage: {
      type: String,
      default: 'Maximum files that can be added to uploader'
    },
    retryErrorMessage: {
      type: String,
      default: 'Please remove the files and try again.'
    },
    httpMethodErrorMessage: {
      type: String,
      default: "This HTTP method is not allowed. Please use either 'put' or 'post' methods."
    },
    showHttpMessages: {
      type: Boolean,
      default: true
    }
  },

  /*
   * The component's data.
   */
  data () {
    return {
      dragging: false,
      items: [],
      itemsAdded: '',
      itemsNames: [],
      itemsSizes: [],
      itemsTotalSize: '',
      formData: '',
      successMsg: '',
      errorMsg: '',
      isLoaderVisible: false,
    }
  },

  methods: {
    // http://scratch99.com/web-development/javascript/convert-bytes-to-mb-kb/
    bytesToSize (bytes) {
      const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
      if (bytes === 0) return 'n/a';
      let i = parseInt(Math.floor(Math.log(bytes) / Math.log(1024)));
      if (i === 0) return bytes + ' ' + sizes[i];
      return (bytes / Math.pow(1024, i)).toFixed(2) + ' ' + sizes[i];
    },

    onChange (e) {
      this.successMsg = '';
      this.errorMsg = '';
      this.formData = new FormData();
      let files = e.target.files || e.dataTransfer.files;
      this.itemsAdded = files.length;
      let fileSizes = 0;
      for (let x in files) {
        if (!isNaN(x)) {
          this.items = e.target.files[x] || e.dataTransfer.files[x];
          this.itemsNames[x] = files[x].name;
          this.itemsSizes[x] = this.bytesToSize(files[x].size);
          fileSizes += files[x].size;
          this.formData.append('items[]', this.items);
        }
      }
      this.itemsTotalSize = this.bytesToSize(fileSizes);
    },

    removeItems () {
      this.items = '';
      this.itemsAdded = '';
      this.itemsNames = [];
      this.itemsSizes = [];
      this.itemsTotalSize = '';
      this.dragging = false;
    },

    onSubmit () {
      this.isLoaderVisible = true;

      if ((typeof this.postMeta === 'string' && this.postMeta !== '') ||
        (typeof this.postMeta === 'object' && Object.keys(this.postMeta).length > 0)) {
        this.formData.append('postMeta', this.postMeta);
      }

      if (typeof this.postData === 'object' && this.postData !== null && Object.keys(this.postData).length > 0) {
        for (var key in this.postData) {
          this.formData.append(key, this.postData[key]);
        }
      }

      if (this.method === 'put' || this.method === 'post') {
        axios({ method: this.method, url: this.postURL, data: this.formData, headers: this.postHeader })
          .then((response) => {
            this.isLoaderVisible = false;
            // Show success message
            if (this.showHttpMessages)
              this.successMsg = response.code + "." + this.successMessagePath;
            // 呼叫父層方法
            this.$emit('uploadSuccess', response)
            this.removeItems();
          })
          .catch((error) => {
            this.isLoaderVisible = false;
            if (this.showHttpMessages)
              this.errorMsg = error.code + "." + this.errorMessagePath;
            // 呼叫父層方法
            this.$emit('uploadError', response)
            this.removeItems();
          });
      } else {
        if (this.showHttpMessages)
          this.errorMsg = this.httpMethodErrorMessage;
        this.removeItems();
      }
    },
  }
}
</script>

<style>
.uploadBox {
  position: relative;
  background: #eee;
  padding: 0 1em 1em 1em;
  margin: 1em;
}

.uploadBox h3 {
  padding-top: 1em;
}

.uploadBox .uploadBoxMain {
  position: relative;
  margin-bottom: 1em;
  margin-right: 1em;
}

/* Drag and drop */
.uploadBox .dropArea {
  position: relative;
  width: 100%;
  height: 300px;
  border: 5px dashed #00adce;
  text-align: center;
  font-size: 2em;
  padding-top: 80px;
}

.uploadBox .dropArea input {
  position: absolute;
  cursor: pointer;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
}
/* End drag and drop */

/* Loader */
.uploadBox .loader {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #fff;
  opacity: 0.9;
}

.uploadBox .loaderImg {
  border: 9px solid #eee;
  border-top: 9px solid #00adce;
  border-radius: 50%;
  width: 70px;
  height: 70px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
/* End Loader */

.uploadBox .errorMsg {
  font-size: 2em;
  color: #a94442;
}

.uploadBox .successMsg {
  font-size: 2em;
  color: #3c763d;
}
</style>
