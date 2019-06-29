<template>
  <section class="vue-cropper-image">
    <div class="upload-img" :style="uploadStyle">
      <template v-if="resultImg">
        <img :src="resultImg" :width="viewWidth || width" :height="viewHeight || height">
        <div class="mask-delete" v-if="!disabled">
          <i class="el-icon-delete" @click.stop="onRemoveImg"></i>
        </div>
      </template>
      <template v-else>
        <i class="el-icon-plus" :style="iconStyle"></i>
        <input ref="inputFile" type="file" @change="onUploadImg" v-if="!disabled" :accept="accept" />
      </template>
    </div>
    <el-dialog :visible.sync="dialogVisible" :close-on-click-modal="false" :before-close="handleBeforeClose" width="800px">
      <div class="crop-dialog-wrapper">
        <div class="crop-img-wrapper">
          <div class="img-wrapper" v-loading="loading" element-loading-text="正在保存中">
            <img :src="cropperImg" style="max-width: 100%" ref="img">
          </div>
          <div class="img-i-group">
            <i class="el-icon-zoom-in" @click="onZoom(0.1)"></i>
            <i class="el-icon-zoom-out" @click="onZoom(-0.1)"></i>
            <i class="el-icon-refresh-right" @click="onRotate"></i>
          </div>
        </div>
        <div class="crop-operate-wrapper">
          <h2>图片裁剪</h2>
          <p class="operate-explain">图片比例为{{width / gcdNum}}:{{height / gcdNum}}</p>
          <div class="operate-button-group">
            <div class="re-upload">
              <el-button size="mini" :disabled="loading">重新上传</el-button>
              <input ref="reInputFile" type="file" @change="onReUploadImg" v-if="!disabled && !loading" :accept="accept" />
            </div>
            <el-button class="btn-save" size="mini" type="primary" :disabled="loading" @click="onCropImg">保存</el-button>
          </div>
        </div>
      </div>
    </el-dialog>
  </section>
</template>

<script>
import Cropper from 'cropperjs'
import 'cropperjs/dist/cropper.min.css'

// 内部上传和预览地址，可根据实际需求替换
const UPLOAD_URL = ''
const PREVIEW_URL = ''

export default {
  name: 'vue-cropper-image',
  props: {
    value: {
      type: String,
      default: ''
    },
    domain: {
      type: String,
      required: true
    },
    accept: {
      type: String,
      default: 'image/*'
    },
    width: {
      type: Number,
      default: 200
    },
    height: {
      type: Number,
      default: 200
    },
    viewWidth: {
      type: Number,
      default: 0
    },
    viewHeight: {
      type: Number,
      default: 0
    },
    rate: {
      type: Number,
      default: 1
    },
    disabled: {
      type: Boolean,
      default: false
    },
    defaultUrl: {
      type: String,
      default: ''
    }
  },
  data () {
    return {
      loading: false,
      dialogVisible: false,
      hasChange: false,
      cropper: null,
      cropperImg: '',
      resultImg: '',
      uploadStyle: {
        width: `${this.viewWidth || this.width}px`,
        height: `${this.viewHeight || this.height}px`
      },
      iconStyle: {
        lineHeight: `${this.viewHeight || this.height}px`
      },
      gcdNum: 1
    }
  },
  watch: {
    dialogVisible (newVal) {
      if (!newVal) {
        this.cropper.destroy()
        this.cropper = null
      }
    },
    value (val) {
      // `${PREVIEW_URL}?key=${val}`公司内部图片预览
      this.resultImg = val.trim() ? `${PREVIEW_URL}?key=${val}` : ''
    },
    defaultUrl (val) {
      this.resultImg = val
    }
  },
  mounted () {
    this.gcdNum = this.gcd(this.width, this.height)
  },
  methods: {
    init () {
      this.cropper = new Cropper(this.$refs.img, {
        aspectRatio: this.width / this.height,
        viewMode: 1,
        cropend: (e) => {
          this.hasChange = true
        },
        zoom: (e) => {
          this.hasChange = true
        }
      })
    },
    // 上传图片
    onUploadImg (event) {
      const img = event.target.files[0]
      this.cropperImg = URL.createObjectURL(img)
      this.dialogVisible = true
      this.$refs.inputFile.value = ''
      this.$nextTick(() => {
        this.init()
      })
    },
    // 重新上传图片
    onReUploadImg (event) {
      this.cropper.destroy()
      const img = event.target.files[0]
      this.cropperImg = URL.createObjectURL(img)
      this.$refs.reInputFile.value = ''
      this.$nextTick(() => {
        this.init()
      })
    },
    // 剪切图片并上传到服务器
    onCropImg () {
      this.loading = true
      let rate = this.rate > 0 ? this.rate : 1
      let canvas = this.cropper.getCroppedCanvas({
        width: this.width * rate,
        height: this.height * rate
      })
      canvas.toBlob(blob => {
        let data = {
          content: blob,
          tenant: 'pampas',
          domain: this.domain
        }
        let header = { 'Authorization': '' }
        // 示例上传接口，非实际项目接口
        this.$request.uploadPic(UPLOAD_URL, data, header).then(res => {
          this.loading = false
          if (res.data.key) {
            this.dialogVisible = false
            this.$emit('input', res.data.key)
            setTimeout(() => {
              this.resultImg = `${PREVIEW_URL}?key=${res.data.key}`
            }, 200)
          } else {
            this.$message.warning('上传图片出现异常！')
          }
        }).catch(err => {
          this.loading = false
          console.log(err)
          this.$message.warning('上传图片出现异常！')
        })
      })
    },
    // 删除图片
    onRemoveImg () {
      this.resultImg = ''
      this.$emit('input', '')
    },
    // 计算最大公约数
    gcd (a, b) {
      if (b === 0) {
        return a
      }
      let r = parseInt(a % b)
      return this.gcd(b, r)
    },
    // 放大、缩放：ratio大于0，放大；小于0，缩小
    onZoom (ratio) {
      if (this.loading) {
        return
      }
      this.cropper.zoom(ratio)
    },
    // 旋转： deg大于0，顺时针；小于0，逆时针
    onRotate (e, deg = 45) {
      if (this.loading) {
        return
      }
      this.hasChange = true
      this.cropper.rotate(deg)
    },
    handleBeforeClose (done) {
      if (this.hasChange) {
        this.$confirm('离开后当前页面信息将不会保留信息，请确认是否离开？', '确认离开?', {
          confirmButtonText: '确认离开',
          cancelButtonText: '留在当前页',
          type: 'warning',
          center: true
        }).then(() => {
          done()
        }).catch(() => {})
      } else {
        done()
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.upload-img {
  position: relative;
  display: inline-block;
  border: 1px dashed #d9d9d9;
  border-radius: 6px;
  text-align: center;
  overflow: hidden;
  cursor: pointer;
  i {
    width: 100%;
    font-size: 28px;
    color: #8c939d;
  }
  input {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    opacity: 0;
    z-index: 1;
    width: 100%;
  }

  &:hover {
    .mask-delete {
      position: absolute;
      top: 0;
      bottom: 0;
      left: 0;
      right: 0;
      background: rgba(#9b9b9b, 0.5);
      z-index: 99;
      border-radius: 6px;
    }
    .el-icon-delete {
      display: block;
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      font-size: 28px;
      line-height: 1;
      color: #fff;
      z-index: 999;
      border: none;
    }
  }
}
.img-wrapper {
  max-width: 100%;
}
.crop-dialog-wrapper {
  display: flex;
  overflow: hidden;
  .img-wrapper {
    height: 300px;
    width: 500px;
    img {
      width: auto;
      max-height: 100%;
    }
  }
  .img-i-group {
    margin-top: 20px;
    display: flex;
    justify-content: center;
    i {
      font-size: 28px;
      margin: 0 20px;
      cursor: pointer;
    }
  }
  .crop-operate-wrapper {
    margin-left: 30px;
    h2 {
      font-size: 16px;
      font-weight: bold;
      margin-top: 50px;
    }
    .operate-explain {
      margin-top: 15px;
    }
    .operate-button-group {
      margin-top: 100px;
    }
    .re-upload {
      display: inline-block;
      position: relative;
      input {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        opacity: 0;
        z-index: 1;
        width: 100%;
      }
    }
    .btn-save {
      margin-left: 20px;
    }
  }
}
</style>
