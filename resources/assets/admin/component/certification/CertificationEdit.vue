<style lang="css" module>
    .container {
        padding: 15px;
    }
    .loadding {
        text-align: center;
        font-size: 42px;
        padding-top: 50px;
    }
    .loaddingIcon {
        animation-name: "TurnAround";
        animation-duration: 1.4s;
        animation-timing-function: linear;
        animation-iteration-count: infinite;
    }
    .image {
        max-width:200px;
        margin-bottom: 10px;
    }
</style>
<template>
        <div :class="$style.container">
          <div class="panel panel-default">
            <div class="panel-heading">
                <router-link type="button" class="btn btn-primary btn-sm" :to="{name: 'certification:users'}">返回</router-link>
            </div>
            <div class="panel-body">
                <!-- 加载动画 -->
                <div v-show="loadding" :class="$style.loadding">
                    <span class="glyphicon glyphicon-refresh" :class="$style.loaddingIcon"></span>
                </div>

                <div class="col-md-6 col-md-offset-3" v-show="!loadding">
                    <div class="form-group">
                        <label>用户名：</label>
                        <input type="text" class="form-control" v-model="certification.username" disabled>
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>真实姓名：</label>
                        <input type="text" class="form-control" v-model="certification.name">
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>手机号：</label>
                        <input type="text" class="form-control" v-model="certification.phone">
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>身份证号：</label>
                        <input type="text" class="form-control" v-model="certification.number">
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>认证类型：</label>
                        <select class="form-control" v-model="certification.type" disabled>
                            <option :value="categroy.name" v-for="categroy in categories">{{ categroy.display_name }}</option>
                        </select>
                    </div>
                    <div class="form-group" v-show="certification.type == 'org'">
                        <label><span class="text-danger">*</span>组织名称：</label>
                        <input type="text" class="form-control" v-model="certification.org_name">
                    </div>
                    <div class="form-group" v-show="certification.type == 'org'">
                        <label><span class="text-danger">*</span>组织地址：</label>
                        <input type="text" class="form-control" v-model="certification.org_address">
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>认证描述：</label>
                        <textarea class="form-control" v-model="certification.desc"></textarea>
                    </div>
                    <div class="form-group">
                        <label><span class="text-danger">*</span>认证附件：</label>
                        <img :src="fileBase64" class="img-responsive" :class="$style.image">
                        <input type="file" @change="uploadAttachment" accept="image/gif,image/jpeg,image/jpg,image/png">
                        <span class="help-block" style="font-size:12px;">附件格式：gif, jpg, jpeg, png； 附件大小：不超过10M</span>
                    </div>
                    <div class="form-group">
                        <button class="btn btn-primary btn-sm" 
                        @click.prevent="updateCertification" data-loading-text="提交中" autocomplete="off" id="edit-btn">确认</button>
                        <div class="pull-right">
                            <span class="text-danger" v-show="message.error">{{ message.error }}</span>
                            <span class="text-success" v-show="message.success">{{ message.success }}</span>
                        </div>
                    </div>
                </div>
            </div>
          </div>
        </div>
</template>

<script>
import request, { createRequestURI } from '../../util/request';
import plusMessageBundle from 'plus-message-bundle';
const PersonalCertificationEdit = {
    data: () => ({
        loadding: true,
        message: {
            error: null,
            success: null,
        },
        categories: {},
        id: null,
        fileBase64: '',
        certification: {
            username: '',
            name: '',
            phone: '',
            number: '',
            files: [],
            org_name: '',
            org_address: '',
            desc: '',
            type:'user',
        },
    }),
    methods: {
        getCertificationCategories () {
          request.get(
            createRequestURI('certification/categories'),
            {validateStatus: status => status === 200}
          ).then(response => {
            this.categories = response.data;
          }).catch(({ response: { data: { errors = ['加载认证详情失败'] } = {} } = {} }) => {
          }); 
        },
        getCertification (id) {
          this.loadding = true;
          request.get(
            createRequestURI('certifications/' + id),
            { validateStatus: status => status === 200 }
          ).then(response => {

            this.loadding = false;
            var data = response.data;
            
            this.certification.username = data.user.name;
            this.certification.name = data.data.name;
            this.certification.phone = data.data.phone;
            this.certification.number = data.data.number;
            this.certification.type = data.certification_name;
            this.certification.desc = data.data.desc;
            this.certification.files = data.data.files;
            this.fileBase64 = '/api/v2/files/' + this.certification.files[0]


            if ( data.certification_name === 'org' ) {
                this.certification.org_name = data.data.org_name;
                this.certification.org_address = data.data.org_address;
            }

          }).catch(response => {
            console.log(response);
          });
        },
        updateCertification (e) {
          $('#edit-btn').button('loading');
          request.patch(
            createRequestURI('certifications/' + this.id),
            { ...this.certification },
            {validateStatus: status => status === 201}
          ).then(({ data: { message: [ message ] = [] } }) => {
            $('#edit-btn').button('reset');
            this.message.success = message;
          }).catch(({ response: { data = {} } = {} }) => {
            $('#edit-btn').button('reset');
            let Message = new plusMessageBundle(data);
            this.message.error = Message.getMessage();
          });
        },
        /**
         * 关闭提示弹层
         */
        offAlert () {
            this.errorMessage = this.successMessage = '';
        },
        uploadAttachment (e) {
            var that = this;
            let file = e.target.files[0]; 
            let param = new FormData();
            param.append('file', file);
            //  设置请求头
            let config = {
                headers: { 
                    'Content-Type': 'multipart/form-data',
                    'Authorization': 'Bearer ' + window.TS.token 
                }
            };
            let reader = new FileReader(); 
            reader.readAsDataURL(file); 
            reader.onload = function(e) {
               that.fileBase64 = e.target.result;
               request.post('/api/v2/files', param, config)
               .then((response) => {
                    const { id: id, message: [message] = [] } = response.data;
                    that.certification.files = [id];
                }).catch((error) => {
                  let Message = new plusMessageBundle(error);
                  this.message.error = Message.getMessage();
                });
            }
        }
    },
    created () {
      this.id = this.$route.params.certification;
      //获取认证栏目
      this.getCertificationCategories();
      //获取认证详情
      this.getCertification(this.id);
    },
};
export default PersonalCertificationEdit;
</script>
