<template>
    <div class="body">
        <div class="container">
            <div :style="serverStatus === 'start' ? '' : 'display:none'">
                <el-space size="large" direction="vertical">
                    <el-card class="box-card">
                        <template #header>
                            <div class="card-header">
                                <el-row class="row-bg" justify="space-between">
                                    <el-col :span="6">正在分享...</el-col>
                                    <el-col :span="6">
                                        <el-button type="danger" size="small" @click="stopServer" plain>取消分享</el-button>
                                    </el-col>
                                </el-row>
                            </div>
                        </template>
                        <el-row class="row-bg">
                            <el-col :span="12">分享链接：{{ settingForm.url }}</el-col>
                            <el-col :span="4">
                                <el-popover placement="left" :width="100" trigger="hover">
                                    <template #reference>
                                        <el-button type="default" size="mini" icon="el-icon-document-copy"
                                            title="复制链接到剪切板" @click="handleClipboard(settingForm.url, $event)">复制链接
                                        </el-button>
                                    </template>
                                    <qrcode-vue :value="settingForm.url"></qrcode-vue>
                                </el-popover>
                            </el-col>
                            <el-col :span="4">
                                <el-button v-if="netInterfaceNames.length > 1" type="default" size="mini"
                                    icon="el-icon-sort" title="切换网卡" @click="changeNetInterface()">切换网卡</el-button>
                            </el-col>
                        </el-row>
                    </el-card>

                    <el-card class="box-card">
                        <template #header>
                            <el-dialog v-model="dialogFormVisible" title="分享一段文本">
                                <el-input type="textarea" :rows="2" :autosize="{ minRows: 2, maxRows: 4 }"
                                    placeholder="请输入内容" v-model="form.text">
                                </el-input>
                                <template #footer>
                                    <el-button type="primary" @click="formSubmit">提交</el-button>
                                </template>
                            </el-dialog>

                            <el-dialog v-model="settingFormVisible" title="设置" style="width: 70%;">
                                <el-row class="row-bg">
                                    <el-col :span="24">
                                        <el-form ref="form" :model="settingForm" label-width="80px">
                                            <el-form-item label="上传路径">
                                                <el-input v-model="settingForm.uploadPath"></el-input>
                                            </el-form-item>
                                            <el-form-item label="服务端口">
                                                <el-input v-model="settingForm.port"></el-input>
                                            </el-form-item>
                                            <el-form-item label="密码认证">
                                                <el-switch v-model="settingForm.authEnable">
                                                </el-switch>
                                            </el-form-item>
                                            <el-form-item v-if="settingForm.authEnable" label="校验密码">
                                                <el-input v-model="settingForm.password" show-password></el-input>
                                            </el-form-item>
                                            <el-form-item label="启用续传">
                                                <el-switch v-model="settingForm.tusEnable" >
                                                </el-switch>
                                            </el-form-item>
                                            <el-form-item v-if="settingForm.tusEnable" label="分片大小">
                                                <el-input v-model="settingForm.chunkSize">
                                                    <template #append><span>M</span></template>
                                                </el-input>
                                            </el-form-item>
                                        </el-form>
                                    </el-col>
                                </el-row>
                                <template #footer>
                                    <el-button type="primary" @click="updateSettingsForm">更新</el-button>
                                    <el-button type="primary" @click="closeSettingsForm">取消</el-button>
                                </template>
                            </el-dialog>

                            <div class="card-header">
                                <el-row class="row-bg">
                                    <el-col :span="12">分享列表</el-col>
                                    <el-col :span="4">
                                        <el-button @click="dialogFormVisible = true" type="default" size="mini"
                                            icon="el-icon-message" title="分享一段文本">分享文本</el-button>
                                    </el-col>
                                    <el-col :span="4">
                                        <el-popconfirm @confirm="removeFileAll()" title="确定要清空所有文件吗？">
                                            <template #reference>
                                                <el-button type="default" size="mini" icon="el-icon-delete"
                                                    title="清空列表">清空列表</el-button>
                                            </template>
                                        </el-popconfirm>
                                    </el-col>
                                    <el-col :span="4">
                                        <el-button @click="onHandlerSetting()" type="default" size="mini"
                                            icon="el-icon-setting" title="设置">设置</el-button>
                                    </el-col>
                                </el-row>
                            </div>
                        </template>

                        <div class="upload-box">
                            <el-upload ref="uploadFile" accept="" drag multiple :show-file-list="false" action="" :http-request="addFiles">
                                <i class="el-icon-upload"></i>
                                <div class="el-upload__text">
                                    拖拽<b>文件</b>或<b>文件夹</b>到此处或点击<em>选择文件</em>，进行分享~
                                </div>
                            </el-upload>
                        </div>

                        <div v-for="file in files" :key="file" class="text item file-item">
                            <el-row class="row-bg" justify="space-between">
                                <el-col :span="5">
                                    <el-tooltip effect="light" placement="top">
                                        <template #content>{{ `由【${file.username}】分享` }}</template>
                                        <span class="username">{{ file.username }}</span>
                                    </el-tooltip>
                                </el-col>
                                <el-col :span="13">
                                    <el-tooltip v-if="file.type === 'text'" effect="light" placement="top">
                                        <template #content>{{ file.intro }}</template>
                                        <span>{{ file.name }}</span>
                                    </el-tooltip>
                                    <span v-if="['directory', 'file'].includes(file.type)">{{ file.name }}</span>
                                </el-col>
                                <el-col :span="6">
                                    <el-button v-if="file.type === 'text'" type="default" size="mini"
                                        icon="el-icon-document-copy" title="复制文本到剪切板"
                                        @click="handleClipboard(file.content, $event)"></el-button>
                                    <el-button v-if="['directory', 'file'].includes(file.type)" type="default" size="mini"
                                        icon="el-icon-search" title="打开文件所在目录" @click="openFile(file.name, $event)">
                                    </el-button>
                                    <el-button type="default" size="mini" icon="el-icon-delete"
                                        @click="() => removeFile(file)"></el-button>
                                </el-col>
                            </el-row>
                        </div>

                        <el-alert v-if="files.length === 0" title="无" :closable="false" type="info" center>
                        </el-alert>
                    </el-card>
                </el-space>
            </div>
            <div :style="serverStatus === 'start' ? 'display:none' : ''">
                <div class="btn-box">
                    <div class="start-btn" @click="startServer">开启服务</div>
                    <div class="start-btn-shadow">
                        <span style="--i:1"></span>
                        <span style="--i:2"></span>
                        <span style="--i:3"></span>
                        <span style="--i:4"></span>
                        <span style="--i:5"></span>
                        <span style="--i:6"></span>
                        <span style="--i:7"></span>
                        <span style="--i:8"></span>
                        <span style="--i:9"></span>
                        <span style="--i:10"></span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import Clipboard from 'clipboard'
import { ElMessage } from 'element-plus'
import QrcodeVue from 'qrcode.vue'

let api = window.api;

let copyClipboard = (text, event) => {
    const clipboard = new Clipboard(event.target, {
        text: () => text
    })
    clipboard.on('success', () => {
        ElMessage.success({ message: '复制成功', type: 'success' });
    })
    clipboard.onClick(event)
}

export default {
    name: 'App',
    data: () => {
        return {
            serverStatus: 'stop',
            qrcode: "",
            files: [],
            netInterfaceNames: [],
            currentNetInterfaceIdx: 0,
            form: {
                text: ''
            },
            settingFormVisible: false,
            settingForm: {
                url: '',
                uploadPath: '',
                port: 5421
            },
            dialogFormVisible: false,
            timer: null
        }
    },
    methods: {
        onHandlerSetting: function () {
            console.log("--onHandlerSetting--")
            this.settingForm = api.getSetting();
            this.settingFormVisible = true;
        },
        updateSettingsForm: function () {
            console.log(this.settingForm)
            api.updateSetting(this.settingForm)
                .then(() => {
                    ElMessage.success("更新成功");
                    this.settingForm = api.getSetting();
                    this.settingFormVisible = false;
                })
                .catch(() => {
                    ElMessage.error("更新失败");
                    this.settingForm = api.getSetting();
                    this.settingFormVisible = false;
                })
        },
        closeSettingsForm: function () {
            this.settingForm = api.getSetting();
            this.settingFormVisible = false;
        },
        formSubmit: function () {
            let text = this.form.text;
            api.addText(text, this.settingForm.ip);
            this.files = api.listFiles();
            this.form.text = '';
            this.dialogFormVisible = false;
        },
        startServer: function () {
            this.settingForm = api.getSetting()
            api.startServer();
        },
        stopServer: function () {
            api.stopServer()
        },
        addFiles: function (params) {
            console.log("addFiles", params)
            let file = { name: params.file.name, path: params.file.path, username: this.settingForm.ip };
            let { success, message } = api.addFile(file);
            if (success) {
                this.files = api.listFiles();
            } else {
                ElMessage.error(message);
            }
        },
        removeFileAll: function () {
            this.files.forEach(f => {
                api.removeFile(f)
            })
            this.files = api.listFiles();
            ElMessage.success({ message: '已清空列表', type: 'success' });
        },
        removeFile: function (file) {
            let removeFiles = this.files.filter((f) => f.name === file.name);
            console.log(removeFiles)
            api.removeFile(removeFiles[0])
            this.files = api.listFiles();
        },
        openFile: function (filename) {
            api.openFile(filename, (err) => {
                ElMessage.error({ message: `文件打开失败 "${err}"`, type: 'error' });
            })
        },
        handleClipboard: function (data, event) {
            copyClipboard(data, event)
        },
        changeNetInterface: function () {
            this.currentNetInterfaceIdx = this.currentNetInterfaceIdx + 1;
            let name = this.netInterfaceNames[this.currentNetInterfaceIdx % this.netInterfaceNames.length];
            let ip = api.getIpAddress(this.currentNetInterfaceIdx);
            api.updateIp(ip).then(() => {
                this.settingForm = api.getSetting()
                ElMessage.success({ message: `切换网卡为 "${name}"`, type: 'success' });
            })
        },
        updatePage() {
            this.serverStatus = api.getServerStatus();
            this.netInterfaceNames = api.getNetInterfaceNames();
            this.files = api.listFiles();
            this.settingForm = api.getSetting()
            console.log("---settingForm--", this.settingForm)
        }
    },
    mounted: function () {
        // document.getElementsByClassName('el-upload__input')[0].webkitdirectory = true
        // 注册事件监听
        api.registryEventListener('server.statusChange', (event) => {
            console.log("---服务状态变更---", event)
            this.serverStatus = event.data.status
        })
        api.registryEventListener('fileDb.listChange', (event) => {
            console.log("---文件列表变更---", event)
            this.files = api.listFiles();
            console.log(this.files)
        })
        this.updatePage()
    },
    beforeUnmount() {
        clearInterval(this.timer)
        this.timer = null
    },
    components: { QrcodeVue }
}
</script>

<style>
body {
    margin: 0;
}

.body {
    font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
    background-image: linear-gradient(to top, #fbc2eb 0%, #a6c1ee 100%);
    background-blend-mode: screen, overlay, hard-light, color-burn, color-dodge, normal;
    background-attachment: fixed;
    background-repeat: no-repeat;
    background-position: 0 100%;
    min-height: 600px;
    position: absolute;
    width: 100%;
    height: 100%;
}

.container {
    max-width: 700px;
    margin: 0 auto;
    padding-top: 18px;
}

.upload-box {
    display: flex;
    justify-content: center;
    align-items: center
}

.el-upload-dragger .el-icon-upload {
    font-size: 46px !important;
    margin: 0 !important;
}

.el-upload-dragger {
    height: 90px !important;
    width: 500px !important;
    margin-bottom: 16px;
}

.file-item {
    margin-bottom: 10px;
}

.username {
    font-size: 12px;
    margin-left: 2px;
    color: #909399;
}

.box-card {
    width: 700px;
}

.row-bg {
    align-items: center;
}

.btn-box {
    width: 150px;
    height: 150px;
}

.el-popover.el-popper {
    min-width: 100px !important;
}

.start-btn {
    text-align: center;
    line-height: 100px;
    width: 100px;
    height: 100px;
    border-radius: 50%;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
    z-index: 99;
    position: absolute;
    top: 150px;
    left: 50%;
    margin-top: -50px;
    margin-left: -50px;
    background-color: #409eff;
    color: #ffffff;
}

.start-btn:hover {
    cursor: pointer;
    background-color: #66b1ff;
}

.start-btn-shadow {
    width: 100px;
    height: 100px;
    display: flex;
    justify-content: center;
    align-items: center;
    position: absolute;
    top: 150px;
    left: 50%;
    margin-top: -50px;
    margin-left: -50px;
    z-index: 1;
}

.start-btn-shadow span {
    z-index: 1;
    position: absolute;
    box-sizing: border-box;
    border: 2px solid #ffffff;
    border-radius: 50%;
    animation: animate 2s linear infinite;
    animation-delay: calc(0.5s * var(--i))
}

.start-btn-shadow:nth-child(2) span {
    border: none;
    background-color: #f5f5f5;
    opacity: 0.8;
}

@keyframes animate {
    0% {
        width: 50px;
        height: 50px;
    }

    50% {
        opacity: 0.5;
    }

    100% {
        width: 200px;
        height: 200px;
        opacity: 0;
    }
}
</style>
