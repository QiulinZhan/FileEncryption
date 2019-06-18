<template>
    <div class="app-container">
        <el-form ref="form" class="form" :model="form" :rules="rules" label-width="120px" hide-required-asterisk>
            <el-form-item label="文件路径" prop="sourcePath">
                <el-input v-model="form.sourcePath">
                    <el-button slot="append" type="primary" @click="select">选择</el-button>
                </el-input>
            </el-form-item>
            <el-form-item label="输出路径" prop="destPath">
                <el-input v-model="form.destPath">
                    <el-button slot="append" type="primary" @click="selectDest">选择</el-button>
                </el-input>
            </el-form-item>
            <el-form-item label="密钥" prop="key">
                <el-input v-model="form.key" :maxlength="32"></el-input>
            </el-form-item>
            <el-form-item label="向量" prop="iv">
                <el-input v-model="form.iv" :maxlength="16"></el-input>
            </el-form-item>
            <el-form-item label="文件类型">
                <el-select
                        v-model="types"
                        placeholder="全部"
                        multiple
                        filterable
                        allow-create
                        :multiple-limit="10"
                        default-first-option>
                    <el-option
                            v-for="item in options"
                            :key="item.value"
                            :label="item.label"
                            :value="item.value">
                    </el-option>
                </el-select>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" @click="onSubmit" :disabled="flag">开始加密</el-button>
            </el-form-item>
        </el-form>
        <el-progress v-if="progressShow" :percentage="percentage" :stroke-width="24" :text-inside="true"></el-progress>
    </div>
</template>

<script>
  import fs from 'fs'
  import Path from 'path'
  // import CryptoJS from 'crypto-js/crypto-js'
  import os from 'os'
  import crypto from 'crypto'
  import Bagpipe from 'bagpipe'

  const { dialog } = require('electron').remote
  export default {
    data() {
      return {
        percentage: 0,
        flag: true,
        progressShow: false,
        rules: {
          sourcePath: [{ required: true, message: '文件路径不能为空', trigger: 'change' }],
          destPath: [{ required: true, message: '输出路径不能为空', trigger: 'change' }],
          key: [{ required: true, message: '密钥不能为空', trigger: 'change' }, {
            min: 32,
            message: '必须是32位字符',
            trigger: 'change'
          }],
          iv: [{ required: true, message: '向量不能为空', trigger: 'change' }, {
            min: 16,
            message: '必须是16位数字',
            trigger: 'change'
          }]
        },
        form: {
          sourcePath: '',
          destPath: '',
          key: '16643216641616321286416643212832',
          iv: '0000000000000000'
        },
        options: [{
          value: 'jpg',
          label: 'jpg'
        }, {
          value: 'png',
          label: 'png'
        }, {
          value: 'jpeg',
          label: 'jpeg'
        }, {
          value: 'gif',
          label: 'gif'
        }, {
          value: 'ico',
          label: 'ico'
        }, {
          value: 'txt',
          label: 'txt'
        }],
        types: [],
        files: [],
        index: 0,
        cpuSize: 10
      }
    },
    mounted() {
      const cpus = os.cpus()
      if (cpus && cpus.length > 0) {
        this.cpuSize = cpus.length
      }
    },
    watch: {
      'form.sourcePath'(v) {
        this.flag = !v
      }
    },
    methods: {
      onSubmit: function() {
        const $this = this
        this.$refs['form'].clearValidate()
        this.$refs['form'].validate(valid => {
          if (valid) {
            this.progressShow = true
            this.percentage = 0
            this.files = []
            this.clearDir(this.form.destPath)
            fs.mkdirSync(this.form.destPath)
            this.walk(this.form.sourcePath)
            // eslint-disable-next-line no-unused-vars
            const size = this.files.length
            // eslint-disable-next-line no-unused-vars
            const iv = new Buffer(this.form.iv.split(''))
            const bagpipe = new Bagpipe(this.cpuSize)
            this.index = 0
            this.files.forEach((file, i) => {
              // eslint-disable-next-line no-unused-vars
              bagpipe.push(this.encFile, size, file, i, iv, function(e, index) {
                $this.index += 1
                $this.percentage = parseInt($this.index / size * 100)
              })
            })
            // this.form.sourcePath = ''
            this.$nextTick(() => {
              this.$refs['form'].clearValidate()
            })
          }
        })
      },
      encFile: function(size, file, i, iv, callback) {
        const newPath = this.form.destPath + file.substring(this.form.sourcePath.length, file.length)
        // eslint-disable-next-line no-unused-vars
        let input
        if (Path.extname(file) === '.png') {
          input = fs.createReadStream(file, { start: 16 })
        } else {
          input = fs.createReadStream(file)
        }
        const output = fs.createWriteStream(newPath)
        if (this.filterExt(file)) {
          // eslint-disable-next-line no-unused-vars
          const cipher = crypto.createCipheriv('aes-256-cbc', this.form.key, iv)
          input.pipe(cipher).pipe(output)
        } else {
          input.pipe(output)
        }
        output.on('finish', () => {
          callback(null, i)
        })
      },
      filterExt(filePath) {
        if (this.types.length === 0) {
          return true
        }
        for (let i = 0; i < this.types.length; i++) {
          if (('.' + this.types[i].toLowerCase()) === Path.extname(filePath)) {
            return true
          }
        }
        return false
      },
      clearDir(path) {
        let files = []
        if (fs.existsSync(path)) {
          files = fs.readdirSync(path)
          files.forEach((file, index) => {
            const curPath = path + '/' + file
            if (fs.statSync(curPath).isDirectory()) {
              this.clearDir(curPath)
            } else {
              fs.unlinkSync(curPath)
            }
          })
          fs.rmdirSync(path)
        }
      },
      walk(path) {
        const list = fs.readdirSync(path)
        list.forEach((file) => {
          const fullPath = path + '/' + file
          const stat = fs.statSync(fullPath)
          if (fullPath === this.form.destPath) {
            return
          }
          if (stat && stat.isDirectory()) {
            const newPath = this.form.destPath + fullPath.substring(this.form.sourcePath.length, fullPath.length)
            fs.mkdirSync(newPath)
            this.walk(fullPath)
          } else {
            this.files.push(fullPath)
          }
        })
      },
      selectDest() {
        dialog.showOpenDialog({ properties: ['openDirectory'] }, (filePaths) => {
          if (filePaths && filePaths.length > 0) {
            const path = filePaths[0]
            const stat = fs.statSync(path)
            if (stat.isDirectory()) {
              this.form.destPath = path
            }
          }
        })
      },
      select() {
        this.progressShow = false
        dialog.showOpenDialog({ properties: ['openDirectory'] }, (filePaths) => {
          // console.log(filePaths)
          if (filePaths && filePaths.length > 0) {
            const path = filePaths[0]
            const stat = fs.statSync(path)
            if (stat.isDirectory()) {
              this.form.sourcePath = path
              this.form.destPath = path + '/new'
            }
          }
        })
      }
    }
  }
</script>

<style scoped>
    .line {
        text-align: center;
    }

    .form {
        margin: 20px 30px 0 0;
    }

    .el-select {
        width: 100%;
    }
</style>


<style lang="scss">
    @import './styles/index.scss'; // 全局自定义的css样式
</style>
