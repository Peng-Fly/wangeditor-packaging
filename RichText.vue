<template>
  <div>
    <div class="btns-wrapper">
      <div class="button" @click="shwoUploadPanelMethods(true)"><label>插入视频</label></div>
      <div class="button" @click="clear"><label>清空</label></div>
      <!-- <button @click="getText">获取html文本</button> -->
      <!-- <button @click="test">test</button> -->
    </div>
    
    <div ref="editor"></div>

    <!-- 上传视频和视频封面图 -->
    <div class="upload-wrapper" v-show="showUploadPanel">
      <div class="upload-video">
        <div class="upload-video-header">
          <div class="upload-video-header-title">插入视频</div>
          <div class="upload-video-header-close" @click="shwoUploadPanelMethods(false)">x</div>
        </div>
        <div class="upload-video-body">
          <div class="upload-video-body-video">
            <div class="upload-video-body-video-item">
              <p>视频文件</p>
              <p class="tips"><span>只能上传mp4文件，且不超过200mb</span></p>
            </div>
            <div class="upload-video-body-video-item">
              <div class="upload-button"><input type="file" ref="video" accept="video/mp4" @change="uploadFile($event, 'video')"> <label>插入视频<span v-if="currentVideoProgress != 0">{{currentVideoProgress}}</span></label></div>
              <div class="preview-video" v-if="currentVideo">
                <video controls>
                  <source :src="currentVideo">
                  您的浏览器不支持视频预览
                </video>
              </div>
            </div>
            <div class="upload-video-body-video-item" style="margin-top:20px;">
              <p>视频封面</p>
            </div>
            <div class="upload-video-body-video-item">
              <div class="upload-button"><input type="file" ref="img" accept="image/jpg,image/png" @change="uploadFile($event, 'img')"> <label>插入图片<span v-if="currentImgProgress != 0">{{currentImgProgress}}</span></label></div>
              <div class="preview-img" v-if="currentImg">
                <img :src="currentImg">
              </div>
            </div>
          </div>
        </div>
        <div class="upload-video-footer">
          <button @click="Uploaded()">取消</button>
          <button @click="insertVideo">确定</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * 使用RichText插件方法(保证掺杂在富文本中的脚本能够执行,主要是视频封面代码)
 * let d = document.createElement('div');
   d.innerHTML = this.data.content;
   let scripts = d.getElementsByTagName('script')[0].innerText;
   console.log(scripts);
   s = document.createElement('script');
   s.innerHTML = scripts;
   document.body.appendChild(s);
 * 
 */
import { baseurl, basePicUrl } from "../../api/api.js";
import E from "wangeditor";

export default {
  data() {
    return {
      editor: null,
      editorContent: "",
      currentVideo: '',
      currentImg: '',
      currentVideoProgress: 0,
      currentImgProgress: 0,
      showUploadPanel: false,     // 显示上传panel
    };
  },
  // props: ['content'],
  props: {
    content: {
      type: String,
      default: ''
    }
  },
  mounted() {
    this.createEditor();
    // console.log(this.editor);
  },
  methods: {
    createEditor() {
      this.editor = new E(this.$refs.editor);
      // 设置z-index
      this.editor.customConfig.zIndex = 1000;
      // 将数据传出去
      this.editor.customConfig.onchange = (html) => {
        this.$emit('content',this.getHtml());
        this.$emit('text',this.getText());
      }
      if (window.location.hostname != "cms.innersect.net") {
        this.editor.customConfig.debug = true;
      }
      // 隐藏网络图片上传tab
      this.editor.customConfig.showLinkImg = false;
      // 关闭粘贴样式过滤
      this.editor.customConfig.pasteFilterStyle = false;
      // 设置表情
      this.setEmotions(this.editor);
      // 上传图片到阿里云OSS
      this.getOSSUploadImgParams(this.editor);
      this.editor.create();
      // 设置内容
      this.editor.txt.html(this.content);
      // 初始化一些内容
      this.add2Html(this.editor.txt.html());
    },
    getOSSUploadImgParams(editor) {
      let key = "activity/" + new Date().getTime() + ".png";
      let xmlhttp = new XMLHttpRequest();
      let serverUrl = baseurl + "/oss/sign?userId=" + key;
      xmlhttp.open("GET", serverUrl, false);
      xmlhttp.send(null);
      let res = eval("(" + xmlhttp.responseText + ")");
      
      console.log(res);
      
      editor.customConfig.customUploadImg = function(files, insert) {
        let formData = new FormData();
        formData.append("OSSAccessKeyId", res.data.accessid);
        formData.append("signature", res.data.signature);
        formData.append("policy", res.data.policy);
        formData.append("key", key);
        formData.append("callback", res.data.callback);
        formData.append("file", files[0]);
        let xhr = new XMLHttpRequest();
        xhr.open("POST", basePicUrl);
        xhr.send(formData);
        xhr.onreadystatechange = () => {
          if (xhr.readyState == 4 && xhr.status == 200) {
            let resultValue = xhr.responseText;
            let data = JSON.parse(resultValue);
            insert(data.link);
          }
        };
      };
    },

    // 上传视频
    uploadFile(e, type) {
      // 清空input的file,保证下次能够触发onchange事件
      console.log(e);
      let temp = e.target.value.split('.');
      let suffix = temp[temp.length - 1];
      let key = "activity/" + new Date().getTime() + '.' + suffix;
      let xmlhttp = new XMLHttpRequest();
      let serverUrl = baseurl + "/oss/sign?userId=" + key;
      xmlhttp.open("GET", serverUrl, false);
      xmlhttp.send(null);
      let res = eval("(" + xmlhttp.responseText + ")");

      console.log(res);

      let formData = new FormData();
      formData.append("OSSAccessKeyId", res.data.accessid);
      formData.append("signature", res.data.signature);
      formData.append("policy", res.data.policy);
      formData.append("key", key);
      formData.append("callback", res.data.callback);
      formData.append("file", e.target.files[0]);
      let xhr = new XMLHttpRequest();
      xhr.open("POST", basePicUrl);
      xhr.onreadystatechange = () => {
        if (xhr.readyState == 4 && xhr.status == 200) {
          console.log(xhr);
          let resultValue = xhr.responseText;
          let data = JSON.parse(resultValue);
          if(type == 'video') {
            this.currentVideo = data.link;
            this.$refs.video.value = '';
          }else {
            this.currentImg = data.link;
            this.$refs.img.value = '';
          }
        }
      };
      xhr.upload.onprogress = (e) => {
        if(type == 'video') {
          let t = parseInt(e.loaded / e.total * 100);
          this.currentVideoProgress = t;
        }else {
          this.currentImgProgress = parseInt(e.loaded / e.total * 100);
        }
      }
      xhr.send(formData);
    },
    // 配置编辑器表情
    setEmotions(editor) {
      editor.customConfig.emotions = [
        {
          title: "表情",
          type: "image",
          content: [
            {
              alt: "[坏笑]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/4d/2018new_huaixiao_org.png"
            },
            {
              alt: "[笑cry]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/4a/2018new_xiaoku_org.png"
            },
            {
              alt: "[馋嘴]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/fa/2018new_chanzui_org.png"
            },
            {
              alt: "[拜拜]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/fd/2018new_baibai_org.png"
            },
            {
              alt: "[右哼哼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/c1/2018new_youhengheng_org.png"
            },
            {
              alt: "[左哼哼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/43/2018new_zuohengheng_org.png"
            },
            {
              alt: "[怒骂]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/87/2018new_zhouma_org.png"
            },
            {
              alt: "[顶]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/ae/2018new_ding_org.png"
            },
            {
              alt: "[微笑]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/e3/2018new_weixioa02_org.png"
            },
            {
              alt: "[偷笑]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/71/2018new_touxiao_org.png"
            },
            {
              alt: "[舔屏]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/3e/2018new_tianping_org.png"
            },
            {
              alt: "[亲亲]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/2c/2018new_qinqin_org.png"
            },
            {
              alt: "[太开心]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/1e/2018new_taikaixin_org.png"
            },
            {
              alt: "[挤眼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/43/2018new_jiyan_org.png"
            },
            {
              alt: "[衰]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/a2/2018new_shuai_org.png"
            },
            {
              alt: "[感冒]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/40/2018new_kouzhao_org.png"
            },
            {
              alt: "[可怜]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/96/2018new_kelian_org.png"
            },
            {
              alt: "[汗]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/28/2018new_han_org.png"
            },
            {
              alt: "[色]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/9d/2018new_huaxin_org.png"
            },
            {
              alt: "[可爱]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/09/2018new_keai_org.png"
            },
            {
              alt: "[钱]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/a2/2018new_qian_org.png"
            },
            {
              alt: "[思考]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/30/2018new_sikao_org.png"
            },
            {
              alt: "[生病]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/3b/2018new_shengbing_org.png"
            },
            {
              alt: "[困]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/3c/2018new_kun_org.png"
            },
            {
              alt: "[互粉]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/86/2018new_hufen02_org.png"
            },
            {
              alt: "[睡]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/e2/2018new_shuijiao_org.png"
            },
            {
              alt: "[并不简单]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/aa/2018new_bingbujiandan_org.png"
            },
            {
              alt: "[害羞]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/c1/2018new_haixiu_org.png"
            },
            {
              alt: "[费解]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/2a/2018new_wenhao_org.png"
            },
            {
              alt: "[挖鼻]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/9a/2018new_wabi_org.png"
            },
            {
              alt: "[悲伤]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/ee/2018new_beishang_org.png"
            },
            {
              alt: "[打脸]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/cb/2018new_dalian_org.png"
            },
            {
              alt: "[抓狂]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/17/2018new_zhuakuang_org.png"
            },
            {
              alt: "[哈哈]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/8f/2018new_haha_org.png"
            },
            {
              alt: "[傻眼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/dd/2018new_shayan_org.png"
            },
            {
              alt: "[晕]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/07/2018new_yun_org.png"
            },
            {
              alt: "[鄙视]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/da/2018new_bishi_org.png"
            },
            {
              alt: "[哼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/7c/2018new_heng_org.png"
            },
            {
              alt: "[哈欠]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/55/2018new_dahaqian_org.png"
            },
            {
              alt: "[泪]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6e/2018new_leimu_org.png"
            },
            {
              alt: "[怒]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/f6/2018new_nu_org.png"
            },
            {
              alt: "[闭嘴]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/62/2018new_bizui_org.png"
            },
            {
              alt: "[疑问]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/b8/2018new_ningwen_org.png"
            },
            {
              alt: "[白眼]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/ef/2018new_landelini_org.png"
            },
            {
              alt: "[吐]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/08/2018new_tu_org.png"
            },
            {
              alt: "[黑线]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/a3/2018new_heixian_org.png"
            },
            {
              alt: "[委屈]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/a5/2018new_weiqu_org.png"
            },
            {
              alt: "[笑而不语]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/2d/2018new_xiaoerbuyu_org.png"
            },
            {
              alt: "[阴险]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/9e/2018new_yinxian_org.png"
            },
            {
              alt: "[嘘]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/b0/2018new_xu_org.png"
            },
            {
              alt: "[嘻嘻]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/33/2018new_xixi_org.png"
            },
            {
              alt: "[爱你]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/f6/2018new_aini_org.png"
            },
            {
              alt: "[吃惊]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/49/2018new_chijing_org.png"
            },
            {
              alt: "[污]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/10/2018new_wu_org.png"
            },
            {
              alt: "[鼓掌]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6e/2018new_guzhang_org.png"
            },
            {
              alt: "[憧憬]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/c9/2018new_chongjing_org.png"
            },
            {
              alt: "[酷]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/c4/2018new_ku_org.png"
            },
            {
              alt: "[失望]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/aa/2018new_shiwang_org.png"
            },
            {
              alt: "[good]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/8a/2018new_good_org.png"
            },
            {
              alt: "[弱]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/3d/2018new_ruo_org.png"
            },
            {
              alt: "[耶]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/29/2018new_ye_org.png"
            },
            {
              alt: "[来]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/42/2018new_guolai_org.png"
            },
            {
              alt: "[握手]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/e9/2018new_woshou_org.png"
            },
            {
              alt: "[加油]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/9f/2018new_jiayou_org.png"
            },
            {
              alt: "[haha]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/1d/2018new_hahashoushi_org.png"
            },
            {
              alt: "[拳头]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/86/2018new_quantou_org.png"
            },
            {
              alt: "[赞]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/e6/2018new_zan_org.png"
            },
            {
              alt: "[ok]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/45/2018new_ok_org.png"
            },
            {
              alt: "[NO]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/1e/2018new_no_org.png"
            },
            {
              alt: "[作揖]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/e7/2018new_zuoyi_org.png"
            },
            {
              alt: "[心]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/8a/2018new_xin_org.png"
            },
            {
              alt: "[伤心]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6c/2018new_xinsui_org.png"
            },
            {
              alt: "[中国赞]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/6d/2018new_zhongguozan_org.png"
            },
            {
              alt: "[广告]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/57/2018new_guanggao_thumb.png"
            },
            {
              alt: "[doge]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/a1/2018new_doge02_org.png"
            },
            {
              alt: "[喵喵]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/7b/2018new_miaomiao_org.png"
            },
            {
              alt: "[二哈]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/22/2018new_erha_org.png"
            },
            {
              alt: "[抱抱]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/42/2018new_baobao_org.png"
            },
            {
              alt: "[摊手]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/62/2018new_tanshou_org.png"
            },
            {
              alt: "[跪了]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/75/2018new_gui_org.png"
            },
            {
              alt: "[吃瓜]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/01/2018new_chigua_org.png"
            },
            {
              alt: "[允悲]",
              src:
                "http://img.t.sinajs.cn/t4/appstyle/expression/ext/normal/83/2018new_kuxiao_org.png"
            }
          ]
        }
      ];
    },
    getHtml() {
      let realHtml = this.editor.txt.html();
      let html = this.add2Html(realHtml);
      return html;
    },
    getText() {
      let text = this.editor.txt.text();
      return text;
    },
    // 改变部分html的样式和事件
    add2Html(html) {
      let ele= document.createElement('div');
      ele.innerHTML = html;
      if(ele.children[0].tagName == 'P') {
        ele.children[0].style.margin = '0px';
      }
      let imgs = ele.getElementsByTagName('img');
      for(let i=0;i<imgs.length;i++) {
        // 不能是表情图片,为图片设置width:100%
        if(!imgs[i].getAttribute('data-w-e')) {
          imgs[i].style.width = '100%';
        }
      }
      /*
      function myOnplay(e) {
        console.log('play',e);
        if(e.target.currentTime == 0) {
          e.target.parentNode.parentNode.children[0].style.opacity = '0';
          console.log('play',e.target.parentNode.parentNode.children[0]);
        }
      }
      function myOnended(e) {
        console.log('ended',e);
        e.target.parentNode.parentNode.children[0].style.opacity = '1';
        console.log('ended',e.target.parentNode.parentNode.children[0]);
      }
      */
      // var eles = document.getElementsByClassName('my-rich-text-video');
      // for(var j=0;j<eles.length;j++) {
      //   var item = eles[j];
      //   item.onplay = function(e) {
      //     console.log('play',e);
      //     if(e.target.currentTime == 0) {
      //       e.target.parentNode.parentNode.children[0].style.opacity = '0';
      //       console.log('play',e.target.parentNode.parentNode.children[0]);
      //     }
      //   };
      //   item.onended = function(e) {
      //     console.log('ended',e);
      //     e.target.parentNode.parentNode.children[0].style.opacity = '1';
      //     console.log('ended',e.target.parentNode.parentNode.children[0]);
      //   };
      // }
      // 将函数注册到文档中
      // let s = document.createElement('script');
      // s.innerHTML += myOnplay;
      // s.innerHTML += myOnended;
      // s.innerHTML += 'var eles = document.getElementsByClassName("my-rich-text-video");'+
      //                 'for(var j=0;j<eles.length;j++) {'+
      //                  ' var item = eles[j];'+
      //                   'item.onplay = function(e) {'+
      //                       'console.log("play",e);'+
      //                       'if(e.target.currentTime == 0) {'+
      //                         'e.target.parentNode.parentNode.children[0].style.opacity = "0";'+
      //                         'console.log("play",e.target.parentNode.parentNode.children[0]);'+
      //                       '}'+
      //                     '};'+
      //                   'item.onended = function(e) {'+
      //                    'console.log("ended",e);'+
      //                    'e.target.parentNode.parentNode.children[0].style.opacity = "1";'+
      //                    ' console.log("ended",e.target.parentNode.parentNode.children[0]);'+
      //                   '};'+
      //                 '}';
      // ele.appendChild(s);
      return ele.innerHTML;
    },
    // 插入视频
    insertVideo() {
      if(this.currentVideo == '') {
        alert('请插入视频');
        return ;
      } 
      // this.editor.cmd.do('insertHTML',`<div style="width:100%;position:relative;">
      //   <div style="width:100%; height:100%; position:absolute; left:0px; top:0px">
      //     <img style="width:100%;height:100%;" src="${this.currentImg}">
      //   </div>
      //   <div style="width:100%;">
      //     <video class="my-rich-text-video" controls name="media" style="width:100%;">
      //       <source src="${this.currentVideo}">
      //       您的设备不支持视频播放
      //     </video>
      //   </div>
      //   <div style="clear:both;"></div>
      // </div><br>`);
      this.editor.cmd.do('insertHTML',`<div style="width:100%;position:relative;">
        <div style="width:100%;">
          <video class="my-rich-text-video" controls name="media" poster="${this.currentImg}" style="width:100%;">
            <source src="${this.currentVideo}">
            您的设备不支持视频播放
          </video>
        </div>
      </div><br>`);
      this.add2Html(this.editor.txt.html());
      this.Uploaded();
    },
    test() {
      let ele= document.createElement('div');
      ele.innerHTML = this.editor.txt.html();
      console.log(ele.children);
    },
    clear() {
      let result = confirm('确定要清空所有文本内容吗?');
      if(result) {
        this.editor.txt.html('');
      }
    },
    // 是否显示上传panel
    shwoUploadPanelMethods(flag) {
      this.showUploadPanel = flag;
    },
    // 放弃上传视频,也可以上传完成后调用
    Uploaded() {
      this.showUploadPanel = false;
      this.currentVideo = '';
      this.currentImg = '';
      this.currentVideoProgress = 0;
      this.currentImgProgress = 0;
    }
  }
};
</script>

<style scoped>
.btns-wrapper {
  display: flex;
}
.btns-wrapper .button {
  width: 60px;
  height: 30px;
  /* padding: 5px; */
  border-bottom: none;
  position: relative;
  box-sizing: border-box;
  font-size: 12px;
  border: 1px solid #cccccc;
  border-bottom: none;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-right: 5px;
}
.btns-wrapper .button input{
  display: inline-block;
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
}
.btns-wrapper .button button {
  display: inline-block;
  width: 100%;
  height: 100%;
}

.upload-wrapper {
  width: 100%;
  height: 100%;
  position: fixed;
  left: 0;
  top: 0;
  background: rgba(0, 0, 0, 0.3);
  z-index: 1001;
}
.upload-wrapper .upload-video {
  width: 50%;
  height: 50%;
  background: #ffffff;
  border-radius: 5px;
  position: absolute;
  left: 50%;
  top: 50%;
  margin-left: -25%;
  margin-top: -25%;
  padding: 20px;
}
.upload-wrapper .upload-video .upload-video-header {
  width: 100%;
  height: 10%;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.upload-wrapper .upload-video .upload-video-header .upload-video-header-title {
  font-size: 18px;
  color: #303133;
}
.upload-wrapper .upload-video .upload-video-header .upload-video-header-close {
  font-size: 16px;
  color: #909399;
  cursor: pointer;
  font-weight: 100;
}
.upload-wrapper .upload-video .upload-video-body {
  width: 100%;
  height: 80%;
  /* background: #cccccc; */
  overflow: scroll;
}
.upload-wrapper .upload-video .upload-video-body .upload-video-body-video-item {
  margin-bottom: 5px;
}

.upload-wrapper .upload-video .upload-video-footer {
  width: 100%;
  height: 10%;
  display: flex;
  justify-content: flex-end;
  align-items: center;
}
.upload-wrapper .upload-video .upload-video-footer button{
  color: #606266;
  padding: 12px 20px;
  font-size: 14px;
  cursor: pointer;
  background: #ffffff;
  border-radius: 5px;
  outline: none;
}
.upload-wrapper .upload-video .upload-video-footer button:nth-child(2){
  margin-left: 10px;
  background: #409EFF;
  border-color: #409EFF;
  color: #ffffff;
}

.tips {
  color: #cccccc;
  font-size: 12px;
  line-height: 200%;
}
.upload-button {
  width: 100px;
  height: 32px;
  font-size: 14px;
  /* padding: 9px 15px; */
  position: relative;
  background: #409EFF;
  color: #ffffff;
  border-radius: 5px;
  display: flex;
  justify-content: center;
  align-items: center;
}
.upload-button input {
  display: inline-block;
  width: 100%;
  height: 100%;
  opacity: 0;
  position: absolute;
  left: 0;
  top: 0;
}
.preview-img {
  width: 50%;
}
.preview-video {
  width: 50%;
}
.preview-img img {
  width: 100%;
  height: auto;
}
.preview-video video {
  width: 100%;
  height: auto;
}
</style>

