<template>
    <div class="diskMenu">
        <div title="" class="cloud__upload"><input name="file" value="upload" @change="uploadFile($event)" type="file" /></div>
        <el-button type="primary" size="small" icon="el-icon-upload">上传</el-button>
        <el-button type="primary" size="small" icon="el-icon-folder-add" @click="addFolder" plain>新建文件夹</el-button>
        <!-- <el-button type="warning" size="small" icon="el-icon-goods">回收站</el-button> -->
        <div v-show="showEditMenu" class="diskMenu__edit">
            <el-button-group>
                <el-tooltip class="item" effect="dark" content="移动到" placement="bottom">
                    <el-button size="small" icon="el-icon-scissors" @click="openDialog()"></el-button>
                </el-tooltip>
                <el-tooltip class="item" effect="dark" content="删除" placement="bottom">
                    <el-button size="small" icon="el-icon-delete" @click="deleteList"></el-button>
                </el-tooltip>
            </el-button-group>
        </div>
        <div v-show="showProgress" class="upload__progress">
            <el-progress :percentage=percent status="success"></el-progress>
        </div>
        <el-dialog title="选择目录" :visible.sync="dialogVisible" :destroy-on-close="true" :close-on-click-modal="false">
            <el-tree :data="treeData" :props="treeProps" :expand-on-click-node="false" @node-click="treeNodeClick">
            </el-tree>
            <div slot="footer" class="dialog-footer">
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="confirmMove">确 定</el-button>
            </div>
        </el-dialog>
    </div>
</template>

<script>
    import Vue from "vue";
    import http from "../http.js";
    import store from '../store.js';
    import { Button, ButtonGroup, Tooltip, Progress, Message, MessageBox, Dialog, Tree } from "element-ui";
    Vue.use(Button)
    Vue.use(ButtonGroup)
    Vue.use(Tooltip)
    Vue.use(Progress)
    Vue.use(Dialog)
    Vue.use(Tree)

    const messageShow = (type, msg, _this) => {
        Message({
            message: msg,
            type: type,
            center: true
        });
        if (_this) {
            setTimeout(function () {
                _this.showProgress = false;
                _this.percent = 0;
            }, 2000)
        }
    }
    const generTreeData = (parId, oriData) => {
        let res = [];
        for(let k = 0; k < oriData.length; k++){
            if(oriData[k].parId == parId){
                let obj = {
                    name: oriData[k].name,
                    _id: oriData[k]._id,
                    pathRoot: oriData[k].pathRoot,
                }
                obj.children = generTreeData(oriData[k]._id, oriData);
                res.push(obj)
            }
        }
        return res
    }
    export default {
        name: 'DiskMenu',
        props: {
            showEditMenu: Boolean,
            curFoldId: String,
            pathRoot: Array,
            checkList: Array
        },
        data() {
            return {
                showProgress: false,
                percent: 0,
                dialogVisible: false, //弹出目录层级
                treeProps: {
                    label: 'name',
                    children: 'children'
                },
                treeData: [{
                    name: '全部目录',
                    _id: '',
                    pathRoot: [],
                    children: []
                }],
                hasClickNode: null
            }
        },
        computed: {},
        methods: {
            uploadFile: function (e) { //上传文件
                const document = e.target;
                const file = document.files[0];
                if (file.size > 10 * 1000 * 1000) {
                    messageShow('error', '文件大小不能超过10M');
                    return
                }
                let _this = this;
                //创建formdata对象
                let formData = new FormData();
                formData.append('file', file);
                formData.append('foldId', this.curFoldId);
                formData.append('pathRoot', this.pathRoot.concat(this.curFoldId));
                let xhr = new XMLHttpRequest();
                xhr.open('post', 'http://localhost:3000/api/uploadfile');
                xhr.setRequestHeader('skyAuth', `aut${store.state.token}`); //涉及到认证，需要自定义header头部
                xhr.setRequestHeader('authId', `aut${store.state.userId}`); //自定义header头部
                xhr.onreadystatechange = function (e) {
                    if (xhr.readyState == 4 && xhr.status == 200) {
                        const response = JSON.parse(xhr.responseText);
                        messageShow(response.status, response.message, _this);
                        _this.$emit('cloud_list_handel', { type: 'uploadOk' });
                        document.value = ""; //需要清除value，否则第二次选择同样文件时无反应
                    }
                    if (xhr.readyState == 4 && xhr.status !== 200) {
                        messageShow('error', '上传失败', _this);
                        document.value = "";
                    }
                };
                xhr.upload.onprogress = function (e) { //进度条
                    if (e.lengthComputable) {
                        _this.percent = e.loaded / e.total * 100;
                    }
                }
                xhr.upload.onerror = function (err) {
                    messageShow('error', '网络连接失败', _this);
                    document.value = "";
                }
                xhr.send(formData);
                _this.showProgress = true;
            },
            addFolder: function () { //新建文件夹
                MessageBox.prompt('', '新建文件夹', {
                    inputPlaceholder: '请输入名称',
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    closeOnClickModal: false,
                    inputValidator: function (val) {
                        if (!val) {
                            return '输入不能为空'
                        }
                        if (!val.match(/^[(\u4e00-\u9fa5)|(a-zA-Z0-9)]+$/)) {
                            return '输入格式有误，只支持中英文、数字'
                        }
                    }
                }).then(({ value }) => {
                    http.postService('addfold', JSON.stringify({ foldname: value, parId: this.curFoldId, pathRoot: this.pathRoot.concat(this.curFoldId)})).then(res => {
                        if (res.status == 'success') {
                            this.$emit('cloud_list_handel', { type: 'addFoldOk' });
                        }
                    }).catch(err => { })
                }).catch(() => { });
            },
            deleteList: function () { //删除操作
                MessageBox.confirm('此操作将永久删除文件(若删除文件夹，则该文件夹的所有数据都会一并删除)，是否继续?', '提示', {
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    type: 'warning'
                }).then(() => {
                    const deleteArr = this.checkList.map(item => {
                        return item._id
                    })
                    http.postService('deletelist', JSON.stringify({ deletelist: deleteArr })).then(res => {
                        if (res.status == 'success') {
                            this.$emit('cloud_list_handel', { type: 'deleteOk' });
                        }
                    }).catch(err => { })
                }).catch(() => { });
            },
            openDialog: function () {
                //查找所有的目录并组装成tree
                http.getService('cloudlist?foldall=').then(res => {
                    var data = [];
                    if (res.status == "success") {
                        this.treeData[0].children = generTreeData('', res.data);
                    }
                }).catch(err => {
                })
                this.hasClickNode = null;
                this.dialogVisible = true;
            },
            treeNodeClick: function(data){
                this.hasClickNode = data;
            },
            confirmMove: function () { //移动操作
                if(!this.hasClickNode){
                    messageShow('warning', '请选择目录');
                    return
                }
                if(this.hasClickNode._id == this.curFoldId){
                    messageShow('warning', '不可选择自身所处目录');
                    return
                }
                this.dialogVisible = false;
                    const moveArr = this.checkList.map(item => {
                        return item._id
                    })
                    const curPath = this.checkList[0].pathRoot;
                    http.postService('movelist', JSON.stringify({ targetId: this.hasClickNode._id, targetPath: this.hasClickNode.pathRoot, movelist: moveArr, curPath})).then(res => {
                        if (res.status == 'success') {
                            this.$emit('cloud_list_handel', { type: 'moveOk' });
                        }
                    }).catch(err => { })
            }
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
    .diskMenu {
        width: auto;
        margin: 10px 0;
        position: relative;
    }

    .diskMenu__edit {
        margin-left: 20px;
        display: inline-block;
    }

    .cloud__upload {
        position: absolute;
        width: 73px;
        height: 32px;
        opacity: 0;
        overflow: hidden;
        cursor: pointer;
    }

    .cloud__upload input {
        width: 100%;
        height: 100%;
        cursor: pointer;
    }

    .upload__progress {
        width: 100%;
        position: absolute;
        top: 37px;
    }
</style>