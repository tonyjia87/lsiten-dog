/*
 * @Author: lsiten 
 * @Date: 2017-11-13 11:31:09 
 * @Last Modified by: lsiten
 * @Last Modified time: 2017-11-17 15:53:18
 */
<template>
  <div>
    <!--  -->
      <div v-show="!user.avatar" class="avatar-box">
        <p>上传狗狗头像</p>
        <p>
             <div v-show="!user.avatar && isUpload" style='width:100px;height:100px;'>
                <x-circle  :percent="precent" :stroke-width="6" :trail-width="6" stroke-color="#85D262" trail-color="#ececec">
                  <span :style="{color: '#85D262'}">{{precent}}%</span>
                </x-circle>
             </div>
          <svg v-show="!user.avatar && !isUpload" class="icon" aria-hidden="true" @click = "choosePhoto">
            <use xlink:href="#dog-shangchuan"></use>
          </svg>
        </p>
      </div>
      <!-- 有头像时显示头像 -->
      <blur :blur-amount=40 :url="user.avatar" :height=180 v-show="user.avatar">
         <div class="center">
            <div v-show="user.avatar && isUpload" style='width:100px;height:100px;'>
              <x-circle :percent="precent" :stroke-width="6" :trail-width="6" stroke-color="#85D262" trail-color="#ececec">
                  <span :style="{color: '#85D262'}">{{precent}}%</span>
                </x-circle>
            </div>
             <img v-show="user.avatar && !isUpload" :src='user.avatar' @click = "choosePhoto">
          </div>
         <div class="username">
            {{user.nickname}}
            <div class="span" >
              <svg  class="icon" aria-hidden="true" @click="showEdit">
                <use xlink:href="#dog-bianji"></use>
              </svg>
            </div>
         </div>
      </blur>
      <card>
         <div slot="content" class="card-demo-flex card-demo-content01">
           <div class="vux-1px-r">
             <span>{{user.score||0}}</span>
            <br/>
            我的积分
          </div>
          <div class="vux-1px-r">
            <span>{{user.videonum||0}}</span>
             <br/>
             我的视频
          </div>
         </div>
      </card>

      <group>
      <cell title="我的视频" link="/user/myvideo" is-link></cell>
      <cell-box>
          <x-button type="warn"  @click.native="logout">退出登录</x-button>          
      </cell-box>
    </group>


    <camera-actionsheet
     :isShow="isShow"
     :menus="menus"
     @input="val=>{isShow=val}"
     :imageOption = "imageOption"
     v-on:success = "cameraSuccess"
     v-on:error = "cameraError" ></camera-actionsheet>


      <!-- 弹出编辑页面 -->
      <div v-transfer-dom class="Edit-from">
        <popup v-model="editVistable" position="bottom" height="100%">
            <div class="popup-top">
                用户编辑
                <span @click="_closeEdit" class="vux-close" style="color:red;"></span>
            </div>
            <div class="popup-content">
              <group>
                <x-input title="昵称"  v-model="user.nickname" placeholder="请输入昵称"></x-input>
                <x-input title="品种"  v-model="user.breed" placeholder="请输入品种"></x-input>
                <x-input title="年龄"  v-model="user.age" placeholder="请输入年龄"></x-input>
                <cell title="性别" value-align="left">
                  <checker v-model="user.sex" default-item-class="radio-item" selected-item-class="radio-item-selected" style="margin-left:10px;">
                    <checker-item value="0">男</checker-item>
                    <checker-item value="1">女</checker-item>
                  </checker>
                </cell>
              </group>
              <div class="btn-sub">
              <x-button type="primary"  @click.native="submitEdit" :show-loading="isSubmit">保存资料</x-button>
              </div>
            </div>
        </popup>
      </div>
  </div>
</template>

<script>
import {
  Card,
  Group,
  Cell,
  CellBox,
  Blur,
  XCircle,
  Popup,
  TransferDom,
  XInput,
  Checker,
  XButton,
  CheckerItem
} from "vux";
import Vue from "vue";
import { mapState } from "vuex";
import store from "store";
import sha1 from "sha1";
import _ from "lodash";
import cameraActionsheet from "../../common/cameraActionsheet";
export default {
  directives: {
    TransferDom
  },
  created() {
    let user = store.get("user");
    this.user = _.assign(this.user, user);
  },
  data() {
    return {
      user: {
        breed: "",
        age: "",
        sex: "0"
      },
      isShow: false,
      menus: {
        album: "相册",
        takePhoto: "拍照"
      },
      timestamp: "",
      avatarData: "",
      isUpload: false,
      precent: 0,
      isAvatar: false,
      editVistable: false,
      isSubmit: false,
      imageOption: {
         destinationType: 0, //base64
      }
    };
  },
  components: {
    Card,
    Group,
    Cell,
    CellBox,
    Blur,
    cameraActionsheet,
    XCircle,
    Popup,
    XInput,
    Checker,
    CheckerItem,
    XButton
  },
  methods: {
    // 编辑页
    showEdit() {
      this.editVistable = true;
    },
    _closeEdit() {
      this.editVistable = false;
    },
    submitEdit() {
      let user = this.user;
      if (user && user.accessToken) {
        let body = {
          accessToken: user.accessToken,
          age: user.age,
          breed: user.breed,
          nickname: user.nickname,
          sex: user.sex
        };
        this.asyncUser(body);
      }
    },
    //退出登录
    logout(){
      this.$store.commit("USER_LOGOUT");
      this.user={};
      this.$router.replace({name:"login"});
    },

    choosePhoto() {
      this.isShow = true;
    },
    cameraSuccess(url) {
      url = "data:image/jpeg;base64," + url; //base64
      this.avatarData = url;
      this.uploadAvatar();
    },
    cameraError(msg) {
      console.log(msg);
    },

    //上传头像
    uploadAvatar() {
      this.isUpload = true;
      this.timestamp = Date.now();
      //获取签名的id
      let signatureUrl = this.config.url.base + this.config.url.imageSignature;
      //cloudinary图床实现
      // let body = {
      //     accessToken: this.user.accessToken,
      //     timestamp: this.timestamp,
      //     type: "avatar",
      //     cloud:"cloudinary"
      //   };
      //七牛图床实现
      let body = {
          accessToken: this.user.accessToken,
          type: "avatar",
          cloud:"qiniu"
        };    
      let param = {
        api: signatureUrl,
        body: body,
        cb: this.cbSignature
      };
      this.$store.dispatch("getSignature", param);
    },
     //获取base64转blob
    dataURItoBlob(dataURI) {
          // convert base64/URLEncoded data component to raw binary data held in a string
          var byteString;
          if (dataURI.split(',')[0].indexOf('base64') >= 0)
              byteString = atob(dataURI.split(',')[1]);
          else
              byteString = unescape(dataURI.split(',')[1]);
          // separate out the mime component
          var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0];
          // write the bytes of the string to a typed array
          var ia = new Uint8Array(byteString.length);
          for (var i = 0; i < byteString.length; i++) {
              ia[i] = byteString.charCodeAt(i);
          }
          console.log(mimeString);
          return new Blob([ia], {type:mimeString});
      },
    //获取完signature的回调
    cbSignature(data) {
      if (data.success) {
         let signature = data.obj.signature;
         let body = new FormData();
         let imageURL = "";
        if(data.obj.key)
        {
          //七牛上传
          let key = data.obj.key;
          body.append("key", key);
          body.append("token", signature);
          body.append("file", this.dataURItoBlob(this.avatarData));
          imageURL = this.config.qiniu.upload;
        }
        else
        {
          //cloudinary上传
          let folder = data.obj.folder;
          let tags = data.obj.tags;
          let key = data.obj.key;
          body.append("folder", folder);
          body.append("signature", signature);
          body.append("timestamp", this.timestamp);
          body.append("tags", tags);
          body.append("public_id", key);
          body.append("api_key", this.config.cloudinary.api_key);
          body.append("resource_type", "image");
          body.append("file", this.avatarData);
          imageURL = this.config.cloudinary.apiBase + this.config.cloudinary.image;
        }      
        let param = {
          api: imageURL,
          body: body,
          header: {
            "Content-Type": "multipart/form-data"
          },
          cb: this.cbuploadAvatar,
          uploading: this.onUploading
        };
        this.$store.dispatch("uploadFile", param);
      }
      else
      {
        this.$vux.toast.show({
            text:data.obj.errorMsg,
            type:"warn"
        })
      }
    },
    //正在上传的回调函数
    onUploading(progressEvent) {
      let that = this;
      let loaded = progressEvent.loaded,
        total = progressEvent.total;
      that.$nextTick(() => {
        that.precent = Math.round(loaded / total * 100);
      });
    },
    //cloudinary上传头像后回掉
    // bytes: 23910
    // created_at: "2017-11-16T10:54:39Z"
    // etag: "3a9f51bd20808ae4229e7791ada631c7"
    // format: "jpg"
    // height: 360
    // placeholder: false
    // public_id: "appDog/avatar/gwultrsdk1nrxgbgqtrx"
    // resource_type: "image"
    // secure_url: "https://res.cloudinary.com/lsiten/image/upload/v1510829679/appDog/avatar/gwultrsdk1nrxgbgqtrx.jpg"
    // signature: "5a96008039c1f8c1ffdb77197b0f68cec577773a"
    // tags: Array[2]
    // type: "upload"
    // url: "http://res.cloudinary.com/lsiten/image/upload/v1510829679/appDog/avatar/gwultrsdk1nrxgbgqtrx.jpg"
    // version: 1510829679
    // width: 640
    //七牛上传的后回调数据
    //hash 
    //key 文件key
    cbuploadAvatar(data) {
      if (data.public_id) {
        this.user.avatar = data.url;
      }
      else
      {
        //七牛上传
        this.user.avatar = this.config.qiniu.fileUrl + "/" +data.key;
      }
      let user = this.user;
      //如果已经登陆
      if (user && user.accessToken) {
        let body = {
          accessToken: user.accessToken,
          avatar: user.avatar
        };
        this.asyncUser(body);
        this.isAvatar = true;
      }
      this.isUpload = false;
      this.precent = 0;
    },
    //异步更新用户信息
    asyncUser(body) {
      let url = this.config.url.base + this.config.url.update;
      let params = {
        api: url,
        body: body,
        cb: this.cbUserUpdate
      };
      this.$store.dispatch("updateUser", params);
    },
    //用户信息更新回调
    cbUserUpdate(data) {
      if (data.success) {
        if(data.obj.isempty)
        {
          this.$vux.toast.show({
              text:data.obj.tips,
              type:"warn"
          })
          return;
        }
        if (this.isAvatar) {
          this.$vux.toast.show({
            text: "头像更新成功",
            type: "success"
          });
          this.isAvatar = false;
        }
        else{
          this._closeEdit();
        }
        this.user = _.assign(this.user, data.obj);
        this.user.sex = this.user.sex.toString();
        this.$store.commit("UPDATE_USER_ALL", this.user); 
      }
      else{
        this.$vux.toast.show({
            text:data.obj.errorMsg,
            type:"warn"
        })
      }
    }
  },
  computed: {
    ...mapState({
      config: state => state.config
    })
  }
};
</script>

<style lang="less">
@import "~vux/src/styles/1px.less";
@import "~vux/src/styles/close.less";
.icon {
  width: 1em;
  height: 1em;
  vertical-align: -0.15em;
  fill: currentColor;
  overflow: hidden;
}
.center {
  display: flex;
  justify-content: center;
  align-items: center;
  padding-top: 20px;
  color: #fff;
  width: 100%;
  font-size: 18px;
}
.center img {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  border: 4px solid #ececec;
}

.username {
  color: #ececec;
  text-align: center;
  font-size: 18px;
  font-weight: bold;
  line-height: 30px;
  margin-top: 10px;
  .span {
    display: inline;
    margin-left: 10px;
    font-size: 18px;
  }
}

.card-demo-flex {
  display: flex;
}
.card-demo-content01 {
  padding: 10px 0;
}
.card-padding {
  padding: 15px;
}
.card-demo-flex > div {
  flex: 1;
  text-align: center;
  font-size: 12px;
}
.card-demo-flex span {
  color: #f74c31;
}

.avatar-box {
  width: 100%;
  height: 150px;
  background-color: #fff;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  p {
    color: #999;
  }
  .icon {
    font-size: 30px;
    width: 60px;
    height: 40px;
    border: 1px solid #999;
    border-radius: 5px;
    margin-top: 15px;
  }
}

.Edit-from {
  .vux-no-group-title {
    margin-top: 0;
  }
  .popup-top {
    background: #fefffe;
    height: 46px;
    line-height: 46px;
    text-align: center;
    .vux-close {
      float: right;
      padding-right: 15px;
      margin-top: 10px;
    }
  }
  .popup_content {
    margin-top: 15px;
  }
  .btn-sub {
    margin-top: 10px;
  }
  .radio-item {
    border: 1px solid #ececec;
    padding: 5px 15px;
  }
  .radio-item-selected {
    border: 1px solid green;
  }
}
</style>
