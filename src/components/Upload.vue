/*jshint -W065 */
<template lang="html">
  <div>
    <form style="display: none" name=theform>
      <a type="radio" name="myradio" ></a> 上传文件名字保持本地文件名字
      <a type="radio" name="myradio" ></a> 上传文件名字是随机文件名, 后缀保留
    </form>

    <h4>您所选择的文件列表：</h4>
    <div id="ossfile"></div>

    <br/>

    <div id="container">
      <a id="selectfiles" href="javascript:void(0);" class='btn'>选择文件</a>
      <a id="postfiles" href="javascript:void(0);" class='btn'>开始上传</a>
    </div>

    <pre id="console">{{message}}</pre>

    <p>&nbsp;</p>
  </div>
</template>

<script>
  import plupload from '../../static/pupload/plupload-2.1.2/js/plupload.full.min.js';

  export default {
    name: 'upload',
    data() {
      return {
        ossFile: '',
        uploadMethod: '',
        accessid: '',
        accesskey: '',
        host: '',
        policyBase64: '',
        signature: '',
        callbackbody: '',
        filename: '',
        key: '',
        expire: 0,
        g_object_name: '',
        g_object_name_type: '',
        now: Date.parse(new Date()) / 1000,
        message: '',
      };
    },
    props: {
      fileId: '',
      beforeUpload: Function,
      onSuccess: Function,
      onError: Function,
      onProgress: Function,
    },
    beforeCreate() {
    },
    mounted() {
      this.$nextTick(() => {
        this.upload();
      });
    },
    methods: {
      sendRequest() {
        const xmlhttp = new XMLHttpRequest();
        // 你的服务端接口地址:  参考demo:http://oss-demo.aliyuncs.com/oss-h5-upload-js-php/
        // 服务端签名后直传文档:  https://help.aliyun.com/document_detail/31926.html
        const serverUrl = 'http://oss-demo.aliyuncs.com/oss-h5-upload-js-php/php/get.php';
        xmlhttp.open('GET', serverUrl, false);
        xmlhttp.send(null);
        return xmlhttp.responseText;
      },
      getSignature() {
        // 可以判断当前expire是否超过了当前时间,如果超过了当前时间,就重新取一下.3s 做为缓冲
        this.now = Date.parse(new Date()) / 1000;
        if (this.expire < this.now + 3) {
          const body = this.sendRequest();
          const e = eval;
          const obj = e(`(${body})`);
          this.host = obj.host;
          this.policyBase64 = obj.policy;
          this.accessid = obj.accessid;
          this.signature = obj.signature;
          this.expire = parseInt(obj.expire, 10);
          this.callbackbody = obj.callback;
          this.key = obj.dir;
          return true;
        }
        return false;
      },
      checkObjectRadio() {
        const tt = document.getElementsByName('myradio');
        for (let i = 0; i < tt.length; i += 1) {
          if (tt[i].checked) {
            this.g_object_name_type = tt[i].value;
            break;
          }
        }
      },
      randomString(len = 32) {
        const chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
        const maxPos = chars.length;
        let pwd = '';
        for (let i = 0; i < len; i += 1) {
          pwd += chars.charAt(Math.floor(Math.random() * maxPos));
        }
        return pwd;
      },
      getSuffix(filename) {
        const pos = filename.lastIndexOf('.');
        let suffix = '';
        if (pos !== -1) {
          suffix = filename.substring(pos);
        }
        return suffix;
      },
      calculateObjectName(filename) {
        if (this.g_object_name_type === 'local_name') {
          this.g_object_name += `${filename}`;
        } else if (this.g_object_name_type === 'random_name') {
          const suffix = this.getSuffix(filename);
          this.g_object_name = this.key + this.randomString(10) + suffix;
        }
        return '';
      },
      getUploadedObjectName(filename) {
        if (this.g_object_name_type === 'local_name') {
          let tmpName = this.g_object_name;
          tmpName = tmpName.replace(`${filename}`, filename);
          return tmpName;
        } else if (this.g_object_name_type === 'random_name') {
          return this.g_object_name;
        }
        return '';
      },
      setUploadParam(up, filename, ret) {
        if (ret === false) {
          this.getSignature();
        }
        this.g_object_name = this.key;
        if (filename !== '') {
          this.calculateObjectName(filename);
        }
        const newMultipartParams = {
          key: this.g_object_name,
          policy: this.policyBase64,
          OSSAccessKeyId: this.accessid,
          // 让服务端返回200,不然，默认会返回204
          success_action_status: '200',
          signature: this.signature,
          callback: this.callbackbody,
        };
        up.setOption({
          url: this.host,
          multipart_params: newMultipartParams,
        });
        up.start();
      },
      upload() {
        const that = this;
        const uploader = new plupload.Uploader({
          runtimes: 'html5,flash,silverlight,html4',
          browse_button: 'selectfiles',
          multi_selection: true,
          container: document.getElementById('container'),
          flash_swf_url: '../../static/pupload/plupload-2.1.2/js/Moxie.swf',
          silverlight_xap_url: '../../static/pupload/plupload-2.1.2/js/Moxie.xap',
          url: 'http://oss.aliyuncs.com',
          filters: {
            mime_types: [{
              title: '允许上传文件类型',
              extensions: 'jpg,gif,png,bmp',
            }],
            // 最大只能上传200mb的文件
            max_file_size: '200mb',
            // 不允许队列中存在重复文件
            prevent_duplicates: true,
          },
          init: {
            PostInit: () => {
              this.ossFile = '';
              document.getElementById('postfiles').onclick = () => {
                that.setUploadParam(uploader, '', false);
                // console.log('...');
                return false;
              };
            },
            FilesAdded: (up, files) => {
              plupload.each(files, (file) => {
                console.log('file_name: ', file.name, ',    ', 'file_size:  ', file.size);
                document.getElementById('ossfile').innerHTML += `<div id="${file.id}">${file.name} (${plupload.formatSize(file.size)})<b></b><div class="progress"><div class="progress-bar" style="width: 0"></div></div></div>`;
                that.message = '';
              });
            },

            BeforeUpload: (up, file) => {
              that.checkObjectRadio();
              that.setUploadParam(up, file.name, true);
            },
            UploadProgress: (up, file) => {
              const d = document.getElementById(file.id);
              d.getElementsByTagName('b')[0].innerHTML = `<span>${file.percent}%</span>`;
              const prog = d.getElementsByTagName('div')[0];
              const progBar = prog.getElementsByTagName('div')[0];
              progBar.style.width = `${2 * file.percent}px`;
              progBar.setAttribute('aria-valuenow', file.percent);
            },
            FileUploaded: (up, file, info) => {
              if (info.status === 200) {
                document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML
                  = `upload to oss success, object name:${this.getUploadedObjectName(file.name)}`;
              } else {
                document.getElementById(file.id).getElementsByTagName('b')[0].innerHTML = info.response;
              }
            },
            Error: (up, err) => {
              console.log('上传失败：', err, that.onError, up);
              if (err.code === -600) {
                that.message = '文件大小超出限制，限制大小为200M';
              } else if (err.status === 403) {
                // console.log('tocken过期，请刷新页面后再上传文件!');
                that.message = '页面失效，请刷新页面后重新上传文件!';
              } else {
                // console.log('文件上传失败，请刷新页面后重新上传文件！');
                that.message = '上传失败，请刷新页面后重新上传文件！';
              }
              if (that.onError) {
                that.onError(err.message, up, err);
              }
            },
          },
        });
        uploader.init();
      },
    },
  };
</script>
<style src="./upload.css"></style>
